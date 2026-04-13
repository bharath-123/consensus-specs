# ePBS devnets testing cases -- P2P Layer

## Bid Gossip Validation

### Happy path

1. Send a bid from a correctly staked, active builder. Single builder in the network. Proposer includes it and builder reveals payload.
2. Send a bid with `bid.value` == 0 from an active builder. Should be valid (builder just wants to build without paying).
3. Have multiple builders in the network sending 1 bid each. The proposer should select the highest value bid ideally (spec leaves selection as proposer policy).

### Multiple bids / rate limiting

1. Send multiple bids from the same builder for the same slot. Only the first should propagate per gossip rule: `[IGNORE] this is the first signed bid seen with a valid signature from the given builder for this slot`.
2. Have multiple builders each send multiple bids. Verify only first per-builder propagates, and only the highest value per (slot, parent_block_hash) propagates.
3. Spam the p2p network with bids from a single builder. Verify DoS prevention: nodes should only forward bids exceeding current highest by a minimum threshold, or forward only at regular intervals (spec note on DoS prevention).
4. Send two bids from different builders with the same value for the same slot and parent_block_hash. Verify the first one seen propagates and the second is ignored (not highest).

### REJECT conditions

1. Send a bid with `bid.execution_payment` > 0. Should be `[REJECT]`ed -- non-zero `execution_payment` indicates a trusted EL payment and MUST NOT be broadcast on gossip.
2. Send a bid from a builder who is not registered (invalid `bid.builder_index`). Should be `[REJECT]`ed.
3. Send a bid from a builder whose deposit epoch is not yet finalized (not yet active per `is_active_builder`). Should be `[REJECT]`ed.
4. Send a bid with `bid.fee_recipient` not matching the proposer's `SignedProposerPreferences` for that slot. Should be `[REJECT]`ed.
5. Send a bid with `bid.gas_limit` not matching the proposer's `SignedProposerPreferences` for that slot. Should be `[REJECT]`ed.
6. Send a bid with an invalid BLS signature. Should be `[REJECT]`ed.
7. Send a bid signed with a different builder's key (valid sig but wrong builder_index). Should be `[REJECT]`ed.

### IGNORE conditions

1. Send a bid with `bid.slot` that is neither the current slot nor the next slot (e.g., 2 slots in the future or a past slot). Should be `[IGNORE]`d.
2. Send a bid before the corresponding `SignedProposerPreferences` for that slot has been seen. Should be `[IGNORE]`d.
3. Send a bid with `bid.value` exceeding the builder's excess balance (`can_builder_cover_bid` returns False). Should be `[IGNORE]`d.
4. Send a bid with `bid.parent_block_hash` referencing an unknown execution payload. Should be `[IGNORE]`d.
5. Send a bid with `bid.parent_block_root` referencing an unknown beacon block. Should be `[IGNORE]`d.
6. Send a bid that is NOT the highest value bid for its (slot, parent_block_hash). Should be `[IGNORE]`d.

## Proposer Preferences Dependency

1. Proposer does NOT broadcast `SignedProposerPreferences` for their slot. Builders cannot submit bids that pass gossip validation for that slot (fee_recipient/gas_limit cannot be matched). Proposer must self-build.
2. Proposer broadcasts `SignedProposerPreferences` with specific fee_recipient and gas_limit. Verify builders can only submit bids matching those exact values.
3. Proposer broadcasts `SignedProposerPreferences` for a slot they are not actually assigned to propose. Should be `[REJECT]`ed at the proposer_preferences gossip level.

## Envelope Delivery (builder reveals payload)

1. Builder submits a valid `SignedExecutionPayloadEnvelope` after their bid is included in a block. Happy path -- PTC should vote `payload_present = True`.
2. Builder submits envelope with `payload.block_hash` not matching `bid.block_hash`. Should be `[REJECT]`ed on gossip.
3. Builder submits envelope with `envelope.builder_index` not matching `bid.builder_index`. Should be `[REJECT]`ed on gossip.
4. Builder submits envelope with invalid signature. Should be `[REJECT]`ed on gossip.
5. Builder submits envelope referencing an unknown `beacon_block_root`. Should be `[IGNORE]`d until block is seen.
6. Builder submits envelope with `envelope.slot` not matching `block.slot`. Should be `[REJECT]`ed.
7. Builder submits two different envelopes for the same block root. Second should be `[IGNORE]`d (first valid envelope wins).
8. Builder submits envelope but the referenced block hasn't passed validation. Should be `[REJECT]`ed.
9. Builder submits envelope with incorrect `state_root`. Should fail `process_execution_payload` verification.
10. Builder submits envelope with incorrect `payload.parent_hash` (doesn't match `state.latest_block_hash`). Should fail processing.
11. Builder submits envelope with incorrect `payload.timestamp`. Should fail processing.

## Payload Withholding

1. Builder sees their bid included in a block but does NOT reveal the payload. PTC should vote `payload_present = False`. Block goes to fork choice as EMPTY.
2. Builder withholds payload because the block was not timely (honest withholding per spec). Verify the builder still forfeits the bid value (payment is recorded at bid inclusion, not payload reveal).
3. Builder reveals payload LATE (after PTC voting deadline at 75% into slot). PTC may vote `payload_present = False` despite eventual delivery. Verify fork choice handles this correctly.

Reviewed till here

## Payment Mechanics

1. Builder's bid is included with `bid.value > 0`. Verify `BuilderPendingPayment` is recorded in `builder_pending_payments[SLOTS_PER_EPOCH + slot % SLOTS_PER_EPOCH]`.
2. Builder reveals payload, attesters accumulate weight. At epoch boundary, verify `process_builder_pending_payments` moves payment to `builder_pending_withdrawals` if quorum (`get_builder_payment_quorum_threshold`) is met.
3. Builder reveals payload but attestation weight does NOT reach quorum. At epoch boundary, the pending payment should be dropped (builder doesn't get paid).
4. Builder withholds payload but `bid.value > 0`. Builder should still forfeit the bid value (payment recorded at bid time). The payment quorum still applies -- if the proposer's block didn't get enough attestation weight, builder may not pay.
5. Proposer is slashed (proposer slashing included in a block). Verify `BuilderPendingPayment` for that proposal slot is cleared -- builder should NOT have to pay a slashed proposer.

## Builder Balance / Staking Edge Cases

1. Builder submits a bid that exactly exhausts their excess balance (balance - MIN_DEPOSIT_AMOUNT - pending_withdrawals == bid.value). Should be valid.
2. Builder submits a bid when they have pending withdrawals that reduce their effective excess balance below the bid value. Should be `[IGNORE]`d on gossip.
3. Builder has multiple pending payments from previous slots. New bid must account for all pending obligations in `can_builder_cover_bid`. Verify a bid that would exceed remaining excess balance is ignored.

## Data Column Sidecars (blobs)

1. Builder reveals payload with blobs and correctly broadcasts `DataColumnSidecar`s. Verify sidecars reference `beacon_block_root` (not block header like pre-Gloas) and pass KZG verification against `bid.blob_kzg_commitments`.
2. Builder reveals payload with blobs but does NOT broadcast data column sidecars. PTC should vote `blob_data_available = False`. Verify fork choice uses `is_payload_data_available` which requires both PTC vote AND local availability.
3. Builder broadcasts sidecars with incorrect KZG proofs. Should be `[REJECT]`ed on the `data_column_sidecar_{subnet_id}` topic.
4. Builder broadcasts sidecars for incorrect subnet. Should be `[REJECT]`ed.
5. Builder broadcasts sidecars where `sidecar.slot` doesn't match the block's slot. Should be `[REJECT]`ed.
6. Builder broadcasts sidecars before the corresponding beacon block is seen. Should be queued for deferred validation per `[IGNORE]` rule.

## Fork Choice Interactions

1. Builder reveals payload and PTC votes `payload_present = True` with > PAYLOAD_TIMELY_THRESHOLD (256) votes. Verify fork choice treats the block as FULL (`is_payload_timely` returns True).
2. Builder reveals payload but PTC votes below threshold. Verify fork choice may treat block as EMPTY depending on `should_extend_payload` logic.
3. Two builders bid for same slot on different parent block hashes (different fork choice branches). Verify gossip correctly handles per-parent_block_hash highest-value filtering.
4. Builder reveals payload on a block that builds on an EMPTY parent (parent payload was withheld). Verify `on_block` uses `block_states` (not `payload_states`) as pre-state and checks `bid.parent_block_hash == parent_bid.parent_block_hash`.
5. Builder reveals payload on a block that builds on a FULL parent. Verify `on_block` uses `payload_states` as pre-state.

## Timing Edge Cases

1. Builder sends a bid for the next slot (not current slot). Verify it passes gossip validation and is available to the next proposer.
2. Builder sends a bid right at slot boundary. Verify clock disparity handling.
3. Builder reveals payload after the attestation deadline (25% into slot) but before PTC deadline (75% into slot). Verify attesters vote `index = 0` (no payload info yet for same-slot) but PTC can still vote `payload_present = True`.

