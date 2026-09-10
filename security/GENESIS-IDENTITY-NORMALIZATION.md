# Genesis identity validation: case-sensitive uniqueness

## Finding

`schemaAddress` and `schemaHex` accept mixed-case hexadecimal identities, but several uniqueness and role-separation checks in `scripts/genesis/ValidatorManager.ts` compare the raw strings.

Ethereum addresses and fixed-size hexadecimal public keys are byte identities; hexadecimal casing does not change the represented bytes. Therefore two spellings of the same identity can bypass string-based uniqueness or role-separation checks while later genesis construction maps them to the same underlying identity.

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

## Proposed fix

Normalize only for identity comparisons (`toLowerCase()`), while preserving the original input for serialization and error messages. The patch in `security/genesis-identity-normalization-fix.patch` implements this narrowly.

## Validation required before upstream submission

Run the repository's normal TypeScript formatting/lint/tests and add regression coverage proving that mixed-case spellings of the same public key/address are rejected by the schema.

This change should be submitted only after those tests pass locally.
