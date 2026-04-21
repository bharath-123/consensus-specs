# FOCIL Network Asynchrony Issue

## Summary

This document describes a potential edge case in the FOCIL (Fork-Choice enforced Inclusion Lists) design where network asynchrony around the view freeze boundary can cause validators to disagree on whether an execution payload satisfies inclusion list constraints, potentially leading to fork choice disagreement.

## Background

In Heze (EIP-7805), FOCIL introduces:
- **Inclusion list committees**: 16 validators per slot who propose transactions that must be included
- **View freeze**: ILs arriving before 75% of slot duration are stored; later arrivals are ignored
- **ePBS integration**: Builders signal which committee members' ILs they included via `inclusion_list_bits`
- **Fork choice enforcement**: Validators independently validate payloads against their local view of ILs

## The Issue

### Scenario

**Timeline:**
- **0s**: Slot N begins
- **~8s (67%)**: Committee members must broadcast ILs by `INCLUSION_LIST_SUBMISSION_DUE_BPS`
- **9s (75%)**: View freeze at `VIEW_FREEZE_CUTOFF_BPS` - ILs arriving after this aren't stored
- **~11s (92%)**: Proposer stops gathering ILs at `PROPOSER_INCLUSION_LIST_CUTOFF_BPS`
- **12s**: Slot N+1 begins, proposer builds block

**Network Asynchrony:**
1. Committee members {A, B, C, D} all broadcast ILs around 8s
2. Builder sees ILs from {A, B, C} before 9s, but D's IL arrives at 9.2s (after view freeze)
3. Proposer also only saw {A, B, C} before their view freeze
4. Some other validators saw all four ILs {A, B, C, D} before their 9s view freeze

### What Happens

#### 1. Builder Creates Bid

Builder sets `inclusion_list_bits` based on what they saw:

```python
# Builder's view: {A, B, C}
bid.inclusion_list_bits = [1, 1, 1, 0, ...]  # Bits 0,1,2 set for A,B,C
```

#### 2. Proposer Validates Bid

From `validator.md`, proposer checks:

```python
# Proposer's local view also has {A, B, C}
is_valid = is_inclusion_list_bits_inclusive(
    get_inclusion_list_store(),
    state,
    slot - 1,
    bid.inclusion_list_bits
)
# Returns True because bid_bits ⊇ local_bits
# Proposer accepts bid
```

#### 3. Validators Receive Payload

From `fork-choice.md:186`, when validators receive the execution payload:

```python
def record_payload_inclusion_list_satisfaction(
    store: Store,
    state: BeaconState,
    root: Root,
    payload: ExecutionPayload,
    execution_engine: ExecutionEngine,
) -> None:
    # Each validator checks against THEIR local view
    inclusion_list_transactions = get_inclusion_list_transactions(
        get_inclusion_list_store(), state, Slot(state.slot - 1)
    )
    is_inclusion_list_satisfied = execution_engine.is_inclusion_list_satisfied(
        payload, inclusion_list_transactions
    )
    store.payload_inclusion_list_satisfaction[root] = is_inclusion_list_satisfied
```

**For validators who saw D's IL:**
```python
# Their get_inclusion_list_transactions() returns transactions from {A, B, C, D}
# But payload only contains transactions from {A, B, C}
# execution_engine.is_inclusion_list_satisfied() returns False
store.payload_inclusion_list_satisfaction[root] = False
```

#### 4. Fork Choice Impact

From `fork-choice.md:228`:

```python
def should_extend_payload(store: Store, root: Root) -> bool:
    # [New in Heze:EIP7805]
    if not is_payload_inclusion_list_satisfied(store, root):
        return False  # Won't extend this payload!

    # ... rest of checks
```

Validators who saw D's IL **won't extend the payload**, causing them to potentially fork off or prefer a different chain.

## Impact Analysis

### Severity Levels

**Low Impact (Normal Operation):**
- Small number of validators have divergent views
- Majority agrees on fork choice
- Chain progresses normally with slightly reduced weight

**Medium Impact (Network Jitter):**
- Significant minority has divergent views
- Temporary disagreement on head
- Self-resolves within 1-2 slots as views converge

**High Impact (Network Partition):**
- Validators split roughly 50/50 on views
- Sustained fork choice disagreement
- Potential chain split or finality delay

### Risk Factors

1. **Geographic Distribution**: Validators in different regions may consistently see ILs at different times
2. **Network Congestion**: During high load, IL propagation times increase
3. **Targeted DoS**: Attacker delays specific ILs to subset of validators
4. **View Freeze Boundary**: The ~1 second window (8-9s) is critical

### Concrete Consequences

1. **Reduced Fork Choice Weight**: Payloads missing ILs from some validators' views get less weight
2. **Short-lived Forks**: Validators with different views may temporarily prefer different chains
3. **Finality Delays**: Disagreement on head prevents FFG finalization
4. **Censorship Risk**: If builder consistently "misses" certain ILs, those transactions get censored

## Protocol Mitigations

### 1. View Freeze Mechanism

From `fork-choice.md:265`:

```python
def on_inclusion_list(store: Store, signed_inclusion_list: SignedInclusionList) -> None:
    inclusion_list = signed_inclusion_list.message

    time_into_slot_ms = ...
    view_freeze_cutoff_ms = get_view_freeze_cutoff_ms(epoch)
    is_before_view_freeze_cutoff = time_into_slot_ms < view_freeze_cutoff_ms

    # Only store ILs arriving before cutoff
    process_inclusion_list(
        get_inclusion_list_store(),
        inclusion_list,
        is_before_view_freeze_cutoff
    )
```

From `inclusion-list.md:78`:

```python
def process_inclusion_list(..., is_before_view_freeze_cutoff: bool) -> None:
    # ... equivocation checks ...

    # Only store `inclusion_list` if it arrived before the view freeze cutoff.
    if is_before_view_freeze_cutoff:
        store.inclusion_lists[key].add(inclusion_list)
```

**Effect**: Limits asynchrony window to ~1 second (8-9s into slot)

### 2. Honest Majority Assumption

Protocol assumes:
- Most honest validators have good network connectivity
- Majority will see the same ILs before view freeze
- Consensus progresses when majority agrees

### 3. Not a Validity Issue

Key distinction:
- Blocks remain **valid** even if some validators mark payload as not satisfied
- Only affects **fork choice weight**, not validity
- Network can still reach consensus if views are similar enough

### 4. Design Notes

From `fork-choice.md:178-184`:

```python
# *Note*: Payloads previously validated as satisfying the inclusion list
# constraints SHOULD NOT be invalidated even if their associated `InclusionList`s
# have subsequently been pruned.

# *Note*: Invalid or equivocating `InclusionList`s received on the p2p network
# MUST NOT invalidate a payload that is otherwise valid and satisfies the
# inclusion list constraints.
```

These notes indicate:
- Late-arriving ILs shouldn't retroactively invalidate payloads
- Equivocations shouldn't invalidate otherwise valid payloads
- Validation is based on view at specific time, not continuously updated

## Design Trade-offs

### The Core Trade-off

**Decentralized Validation vs. Consistency**

| Approach | Pros | Cons |
|----------|------|------|
| **Current: Local Views** | • No trusted party<br>• Each validator independently validates<br>• Resistant to manipulation | • Asynchrony causes disagreement<br>• Potential fork choice splits<br>• Complexity in edge cases |
| **Alternative: Global View** | • All validators see same data<br>• Consistent fork choice<br>• Simpler reasoning | • Requires synchrony assumption<br>• Single point of failure<br>• Easier to manipulate |

### Why Local Views?

The protocol chooses local views because:

1. **Trustlessness**: No validator trusts another's view
2. **Byzantine Resistance**: Even if some validators lie about ILs, others validate independently
3. **Censorship Resistance**: Can't force all validators to ignore an IL
4. **Fork Choice Safety**: Better to have reduced weight than accept invalid payloads

The protocol essentially says:

> *"We accept that validators might disagree slightly on whether a payload satisfies constraints, but as long as the majority of honest validators have similar views, consensus progresses normally. The alternative—trusting a single view—introduces worse failure modes."*

## Potential Attacks

### 1. Strategic IL Withholding

**Attack**: Builder deliberately delays certain ILs to subset of network

**Goal**: Cause validators to disagree on fork choice

**Mitigation**:
- View freeze limits window
- Equivocation detection catches double-ILs
- Multiple committee members (16) increases redundancy

### 2. Network Partition

**Attack**: Attacker partitions network to give different validators different views

**Goal**: Cause chain split or finality delay

**Mitigation**:
- Requires significant network control
- Honest majority in each partition heals split
- Time-limited (1 second window)

### 3. Geographic Exploitation

**Attack**: Exploit latency differences between regions

**Goal**: Consistently cause certain validators to miss ILs

**Mitigation**:
- Randomized committee selection
- Geographic diversity of honest validators
- P2P gossip propagation

## Recommendations

### For Protocol Researchers

1. **Quantify Impact**: Model the probability and severity of view divergence under various network conditions
2. **Timing Analysis**: Evaluate if view freeze timing (75%) is optimal
3. **Committee Size**: Analyze if 16 members provides sufficient redundancy
4. **Fallback Mechanisms**: Consider whether additional consensus layer coordination is needed

### For Implementers

1. **Aggressive IL Propagation**: Prioritize IL gossip propagation before view freeze
2. **Peer Selection**: Maintain diverse peer connections to reduce view divergence
3. **Monitoring**: Track `payload_inclusion_list_satisfaction` metrics to detect systematic issues
4. **Logging**: Record view divergence events for post-mortem analysis

### For Node Operators

1. **Network Quality**: Ensure low-latency, high-bandwidth connections
2. **Peer Diversity**: Connect to geographically diverse peers
3. **Clock Sync**: Maintain accurate time synchronization (view freeze is time-based)
4. **Monitoring**: Watch for fork choice disagreements around IL-containing blocks

## Open Questions

1. **What percentage of validators can have divergent views before consensus breaks?**
   - Needs formal analysis and simulation

2. **How often does this occur in practice?**
   - Requires testnet and mainnet data collection

3. **Should view freeze timing be adjustable?**
   - Trade-off between asynchrony window and IL gathering time

4. **Are additional safety mechanisms needed?**
   - E.g., attestation-based IL confirmation, extended view freeze for certain conditions

5. **How does this interact with MEV and builder behavior?**
   - Do builders have incentive to strategically withhold ILs?

## Conclusion

The FOCIL network asynchrony issue is an inherent consequence of the design choice to use local, view-based validation rather than global consensus on inclusion lists. While this introduces the possibility of validators disagreeing on fork choice, it provides stronger trustlessness and censorship resistance properties.

The protocol mitigates this through view freeze timing, honest majority assumptions, and treating satisfaction as a fork choice weight factor rather than a validity condition. However, under adverse network conditions or targeted attacks, this could lead to fork choice disagreement or finality delays.

Further analysis, simulation, and real-world testing are needed to quantify the practical impact and determine if additional safeguards are necessary.

## References

- **specs/heze/fork-choice.md**: `record_payload_inclusion_list_satisfaction` (line 186)
- **specs/heze/inclusion-list.md**: `get_inclusion_list_transactions`, `process_inclusion_list`
- **specs/heze/validator.md**: Proposer IL validation with `is_inclusion_list_bits_inclusive`
- **specs/heze/beacon-chain.md**: `ExecutionPayloadBid` structure
- **EIP-7805**: Fork-Choice enforced Inclusion Lists specification
