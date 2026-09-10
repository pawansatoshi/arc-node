# EIP-7702 transaction-pool authority-throttle finding

## Status

**Candidate security/liveness issue — local reproduction required before upstream submission.**

## Affected dependency

Arc's workspace pins the Reth v2.2.0 family, including the transaction-pool implementation. Arc's custom transaction pool is a wrapper around Reth's `Pool` and `EthTransactionValidator`.

The upstream Reth issue [#25909](https://github.com/paradigmxyz/reth/issues/25909) documents a targeted transaction-inclusion DoS in the Reth v2.2.0 transaction pool caused by EIP-7702 authorization authorities being tracked in the pool before transaction execution semantics are known.

## Why Arc is exposed

Arc enables the Prague hard fork at timestamp 0, so EIP-7702/set-code transactions are active. Arc's `ArcPoolBuilder` constructs the production pool with Reth's transaction-pool configuration and wraps the standard Ethereum validator rather than replacing Reth's authority accounting.

## Reproduction gate

Do not treat this as a confirmed Arc vulnerability until a local Arc node demonstrates the victim-throttling effect.

1. Start a local Arc node with Prague active.
2. Use a victim EOA whose current nonce is greater than an old EIP-7702 authorization nonce.
3. Build a valid EIP-7702 type-4 transaction containing the stale authorization and submit it externally.
4. Verify that the poisoning transaction is accepted and remains resident in the txpool.
5. Submit two ordinary victim transactions so the second transaction requires a pending transaction/nonce gap.
6. Observe whether the second transaction is rejected or throttled solely due to the stale authority being tracked by the pool.
7. Remove/expire the poisoning transaction and verify normal victim transaction admission returns.

## Expected security property

A stale authorization that is guaranteed to be a no-op at execution must not reserve or consume delegated-account transaction-pool capacity for the named authority.

## Mitigation direction

Do not solve this only by increasing `max_inflight_delegated_slot_limit`. The security property is that stale/non-executable authorizations must not poison authority tracking. A mitigation should gate delegated-account throttling on execution-relevant authorization validity, including the authority's current nonce and the authorization chain-id rules.

## Evidence

- Arc `Cargo.toml` pins the Reth v2.2.0 dependency family.
- `crates/execution-txpool/src/pool.rs` constructs a Reth `Pool` and uses `TransactionValidationTaskExecutor` with Arc's validator wrapper.
- `crates/execution-txpool/src/validator.rs` delegates transaction validation to `EthTransactionValidator` and explicitly processes EIP-7702 authorization lists for denylist checks.
- Arc hardfork configuration activates Prague at timestamp 0.
- Reth issue #25909 documents the authority-throttle behavior and default delegated slot limit of 1.

## Testing policy

Reproduce only against a local environment, not Arc Public Testnet. Upstream submission should include the exact Arc commit/version, environment, signed test transactions, observed errors, and a regression test demonstrating that the stale authorization no longer affects victim transaction admission.
