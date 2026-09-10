# EIP-7702 transaction-pool authority-throttle finding

## Status

**Candidate security/liveness issue — local reproduction required before upstream submission.**

## Affected dependency

Arc's workspace pins the Reth v2.2.0 family, including the transaction-pool implementation. Arc's custom transaction pool is a wrapper around Reth's `Pool` and `EthTransactionValidator`.

The upstream Reth issue [#25909](https://github.com/paradigmxyz/reth/issues/25909) documents a targeted transaction-inclusion DoS in the Reth v2.2.0 transaction pool caused by EIP-7702 authorization authorities being tracked in the pool before transaction execution semantics are known.

## Why Arc is exposed

Arc enables the Prague hard fork at timestamp 0, so EIP-7702/set-code transactions are active. Arc's `ArcPoolBuilder` constructs the production pool with Reth's transaction-pool configuration and wraps the standard Ethereum validator rather than replacing Reth's authority accounting.

The relevant Arc path is:

- `crates/execution-txpool/src/pool.rs`
- `crates/execution-txpool/src/validator.rs`
- Reth v2.2.0 `transaction-pool`

## Expected attack model

An attacker obtains an old/stale EIP-7702 authorization tuple naming a victim authority and submits a type-4 transaction containing that authorization. The authorization can be a guaranteed execution no-op because EIP-7702 execution checks the authority nonce and chain id before applying delegation.

Reth v2.2.0 nevertheless records recovered authorities for pool throttling before execution. With the default delegated in-flight slot limit of 1, the victim can become subject to the delegated-account transaction throttle even though the victim's on-chain account state is unchanged.

This can prevent nonce-gapped/pipelined victim transactions from entering the pool while the poisoning transaction remains resident.

## Arc-specific reproduction plan

1. Start a local Arc/Reth execution node with Prague active (Arc's genesis configuration already activates Prague at timestamp 0).
2. Fund a victim EOA and ensure its normal account nonce is ahead of a previously signed EIP-7702 authorization nonce.
3. Construct a valid type-4 transaction containing that stale authorization and submit it as an external transaction.
4. Confirm the transaction is accepted into the pool.
5. Submit two ordinary transactions from the victim with consecutive nonces (or submit a nonce-gapped second transaction while the first is pending).
6. Observe whether the second victim transaction is rejected/throttled solely because the stale authority is present in the pool.
7. Repeat after the poisoning transaction leaves the pool and confirm the victim returns to normal behavior.

## Important validation rule

Do not test this against Arc Public Testnet. Reproduce only against a local node, consistent with Arc's security-testing guidance.

## Potential mitigation direction

The safest mitigation should remove stale/non-executable authorization tuples from delegated-account throttling rather than merely raising the delegated slot limit. At minimum, authority accounting should incorporate execution-relevant authorization validity (current authority nonce and acceptable chain id) before applying delegated-account inclusion limits.

Any concrete Arc patch should first be validated against Reth's transaction-pool behavior and covered by a regression test proving that a stale authorization cannot throttle ordinary victim transactions.

## Evidence

- Arc workspace pins Reth v2.2.0 dependencies in `Cargo.toml`.
- Arc's `ArcPoolBuilder` builds a Reth `Pool` using the standard Reth transaction validation stack.
- Arc's hardfork configuration activates Prague at timestamp 0.
- Upstream Reth issue #25909 documents the EIP-7702 authority-throttle behavior and its default limit of 1.

## Conclusion

This is a **credible Arc exposure**, but it should remain a candidate until an Arc-local end-to-end PoC demonstrates the victim throttling behavior. The PoC should be the gate for opening an upstream security report or submitting a mitigation PR.
