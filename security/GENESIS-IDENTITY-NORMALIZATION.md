# Genesis identity validation: case-sensitive identity checks

## Finding

`schemaAddress` and `schemaHex` accept mixed-case hexadecimal identities, but several identity checks in `scripts/genesis/ValidatorManager.ts` compare raw hexadecimal strings.

Ethereum addresses and fixed-size hexadecimal public keys represent byte identities; hexadecimal casing does not change the represented bytes. Therefore two spellings of the same identity can bypass string-based uniqueness or role-separation checks while later genesis construction maps them to the same underlying identity.

Affected checks include:

- validator public-key uniqueness
- validator-registerer uniqueness
- controller uniqueness across validators
- operator/role vs proxy-admin separation
- the hard-coded ValidatorRegistry proxy-address comparison

## Example

The same address can be supplied as:

```text
0xAbCDEF0123456789abcdef0123456789ABCDEF01
0xabcdef0123456789ABCDEF0123456789abcdef01
```

A raw `Set` / `===` comparison treats these as different strings even though the decoded address bytes are identical.

## Impact

This is a genesis/configuration integrity issue. It does not by itself grant a permissionless attacker runtime privileges, because the genesis configuration is operator-controlled. The risk is that a malformed or adversarial configuration can pass validation while assigning the same on-chain identity to multiple roles or entries that the validator is intended to reject as duplicates.

## Fix

Normalize hexadecimal identities only at comparison boundaries (`toLowerCase()`), while preserving the original input for serialization and error messages. Proxy-admin comparisons normalize both operands in the shared `enforceOperatorsNotProxyAdmin` helper.

## Regression coverage

`tests/unit/validator-manager-genesis-validation.test.ts` now covers mixed-case collisions for public keys, controllers, validator registerers, and proxy-admin/operator separation.

Before upstream submission, run the repository's normal TypeScript formatting, linting, and unit-test commands and attach the passing CI result to the pull request.
