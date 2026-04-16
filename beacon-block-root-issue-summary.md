# Beacon Block Root Issue with ePBS (EIP-4788)

## Background

**EIP-4788** maintains an on-chain ring buffer (system contract) that stores `parent_beacon_block_root` for each execution block. This allows smart contracts to prove CL state (slashings, deposits, withdrawals, etc.) against beacon block roots without trusting an oracle.

In the current protocol, every slot with an execution payload writes its parent beacon block root into the EIP-4788 contract. Since every beacon block carries an execution payload today, every beacon block root eventually appears in the contract.

## The Problem

ePBS decouples beacon blocks from execution payloads. A beacon block can be "empty" — it contains a bid but no execution payload. The payload is delivered separately by the builder and may never arrive (builder withholding, missed slot, etc.).

When a payload is missing, no EL state transition happens for that slot, and **no beacon block root is written to the EIP-4788 contract**.

### Concrete Example

```
Slots:    B0   B1   B2   B3   B4
Payloads: E0   E1   --   --   E4
```

- B2 and B3 have empty payloads (builder didn't deliver).
- E4 is the first full payload after the gap.
- E4 writes `parent_beacon_block_root = B3's parent root` into EIP-4788.
- **B1 and B2's block roots are never written to the EIP-4788 contract**, even though B1 was a fully valid block with CL operations (slashings, deposits, etc.).

## Why This Matters

### 1. Longer and more expensive Merkle proofs

Currently, proving a withdrawal or CL operation against EIP-4788 requires ~16 sibling hashes (block root -> body root -> execution payload -> withdrawals).

Without the block root on-chain, the proof must go through an indirect path:
- Latest available block root -> state root -> `block_roots[]` array -> target block root -> operation

This requires ~36 sibling hashes — **roughly double the proof size and gas cost**.

### 2. Contracts relying on EIP-4788 break or degrade

Protocols that prove CL events (e.g., decentralized staking pools proving deposits/withdrawals, Lido, Rocket Pool, EigenLayer) currently assume every beacon block root is available in the contract. With ePBS, a full block's root may be absent if the *next* block's payload is empty.

### 3. No Merkle path to execution payload

The bid in the beacon block does not contain an `execution_payload_root` (it can't — some payload fields like withdrawals aren't known at bid time). This means there's no direct Merkle path from a beacon block root to the execution payload contents. The `block_hash` is included in the bid, but that's an RLP hash, not an SSZ root usable for Merkle proofs.

## Potential Solutions Discussed

### Option A: Backfill missed beacon roots when executing the next full payload

When E4 executes, the EL client would also execute the beacon root updates for B2 and B3 in their own execution contexts (with the correct timestamps), then execute E4 — all atomically.

- **Pro**: Restores full EIP-4788 coverage.
- **Con**: Requires engine API changes to pass a list of missed block times + roots. EL-heavy change.

### Option B: Leave it as-is

The beacon block roots are still on-chain in the CL `state.block_roots` array. You can prove against them via a longer Merkle path through the state root of a later block.

- **Pro**: No EL changes needed. Simpler.
- **Con**: Proofs become longer (~36 vs ~16 hashes), more expensive, and more complex. Inconsistent behavior — some block roots are directly available, others require indirection.

### Option C: Add `execution_payload_root` to the bid

Including an SSZ root of the execution payload in the bid would restore a direct Merkle path from beacon block root to payload contents.

- **Pro**: Makes payload contents provable via the beacon block root.
- **Con**: Not all payload fields are known at bid time (e.g., withdrawals are determined by the CL after the bid). However, potuz notes this could work for the payload itself (not the envelope), and it's a CL-only change.

## Current Status

- The issue was raised by pk910 during ePBS devnet testing.
- potuz leans toward the current behavior being acceptable for most use cases, since proofs can go through `state.block_roots`.
- dgusakov (Lido) confirmed this is problematic — they prove exact events in exact blocks (e.g., withdrawals), and requiring variable-length indirection paths is a real issue.
- The group agreed this should be raised at ACD (All Core Devs).
- Adding `execution_payload_root` to the bid is seen as a valuable improvement regardless.

## Key Takeaway

The core tension is between **simplicity** (leave it, proofs get longer) and **compatibility** (fix it, preserve current proof assumptions). The proof cost difference is significant: 16 vs 36 sibling hashes. For protocols doing high-volume on-chain verification of CL state, this is a material cost increase.
