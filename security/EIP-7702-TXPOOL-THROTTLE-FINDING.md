# EIP-7702 transaction-pool authority-throttle finding

## Status

Candidate security/liveness issue. Local Arc reproduction is required before upstream submission.

## Why this is relevant to Arc

Arc's workspace pins the Reth v2.2.0 dependency family. Arc's custom transaction pool builds a Reth `Pool` around the standard Ethereum transaction validator, and Arc activates Prague at timestamp 0, making EIP-7702/set-code transactions active.

Reth issue #25909 documents a targeted transaction-inclusion DoS in Reth v2.2.0: recovered EIP-7702 authorities can be tracked for pool throttling even when the authorization will be a no-op at execution. The documented default delegated in-flight slot limit is 1.

## Arc-specific reproduction gate

This must be reproduced on a local Arc node before calling it a confirmed Arc vulnerability.

1. Start local Arc with Prague active.
2. Use a victim EOA whose current nonce is ahead of an old EIP-7702 authorization nonce.
3. Submit a valid type-4 transaction containing that stale authorization.
4. Verify it remains resident in the transaction pool.
5. Submit ordinary victim transactions with consecutive nonces and observe whether later admission is rejected/throttled because the stale authority is tracked.
6. Remove/expire the poisoning transaction and verify normal admission resumes.

## Expected property

A stale authorization that is guaranteed to be a no-op during EIP-7702 execution must not reserve delegated-account transaction-pool capacity for its named authority.

## Mitigation direction

Do not merely increase `max_inflight_delegated_slot_limit`. Gate authority accounting/throttling on execution-relevant authorization validity, including authority nonce and chain-id rules, then add a regression test.

## Evidence

- Arc pins Reth v2.2.0 in `Cargo.toml`.
- `crates/execution-txpool/src/pool.rs` constructs the production Reth transaction pool.
- `crates/execution-txpool/src/validator.rs` wraps `EthTransactionValidator` and handles EIP-7702 authorization lists.
- Arc's hardfork configuration activates Prague at timestamp 0.
- Reth #25909 documents the authority-throttle behavior in v2.2.0.

## Testing policy

Reproduce only against a local environment, not Arc Public Testnet. Upstream submission should include the exact Arc commit, environment, transaction fixtures, observed pool errors, and a regression test.
