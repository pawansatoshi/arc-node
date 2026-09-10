# EIP-7702 transaction-pool authority-throttle finding

## Status

**Candidate security/liveness issue — local reproduction required before upstream submission.**

## Why this is relevant to Arc

Arc's workspace pins the Reth v2.2.0 dependency family. Arc's custom transaction pool builds a Reth `Pool` around the standard Ethereum transaction validator, and Arc activates Prague at timestamp 0, making EIP-7702/set-code transactions active.

Reth issue [#25909](https://github.com/paradigmxyz/reth/issues/25909) documents a targeted transaction-inclusion DoS in Reth v2.2.0: recovered EIP-7702 authorities can be tracked for pool throttling even when the authorization will be a no-op at execution. The documented default delegated in-flight slot limit is 1.

## Arc-specific reproduction gate

This must be reproduced on a **local Arc node** before calling it a confirmed Arc vulnerability.

1. Start local Arc with Prague active.
2. Use a victim EOA whose current nonce is ahead of an old EIP-7702 authorization nonce.
3. Construct a valid type-4 transaction containing that stale authorization and submit it as an external transaction.
4. Verify the transaction is accepted and remains resident in the txpool.
5. Submit ordinary victim transactions with consecutive nonces and observe whether a later victim transaction is rejected/throttled solely because the stale authority is tracked by the pool.
6. Remove the poisoning transaction and verify normal admission resumes.

## Expected property

A stale authorization that is guaranteed to be a no-op during EIP-7702 execution must not reserve delegated-account transaction-pool capacity for its named authority.

## Mitigation direction

Do not merely raise `max_inflight_delegated_slot_limit`. The stronger fix is to ensure authority accounting only applies to authorizations that are execution-relevant for the current state, including the authority nonce and authorization chain-id rules, followed by an Arc regression test.

## Evidence

- Arc pins Reth v2.2.0 in `Cargo.toml`.
- `crates/execution-txpool/src/pool.rs` constructs the production Reth transaction pool.
- `crates/execution-txpool/src/validator.rs` wraps `EthTransactionValidator` and processes EIP-7702 authorization lists.
- Arc's hardfork configuration activates Prague at timestamp 0.
- Reth #25909 documents the authority-throttle behavior in v2.2.0.

## Testing policy

Use only a local environment for security reproduction, not Arc Public Testnet. An upstream report should contain the exact Arc commit, environment, transaction fixtures, reproduction steps, observed pool errors, and a regression test.
