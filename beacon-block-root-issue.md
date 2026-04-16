
pk910 — 16/03/2026, 09:40
I'm not sure how to check general contract compatibility, but I've played with spamoor to create a transaction per block that just logs out the last 10 blockhashes & beacon roots from both system contracts.  That should be enough to catch differences.. But as both contracts update the state, I guess any mismatch would cause a fork due to mismatching state roots anyway.
While playing around with the system contracts, I noticed another problem:
When execution blocks are missed or reorged out, the beacon block root of the parent block never gets added to the EIP-4788 (beacon block roots) contract. There is no system call that adds parents of empty blocks to the history. 
That's potentially breaking contracts that try to prove stuff that happened in the parent block (like slashings or deposits for decentralized staking pools)
I think we need to find a fix for this, so even parents of blocks with empty payloads make it into the EIP-4788 system contract history. 


potuz — 16/03/2026, 15:36
I suspect the lucid guys like this feature from this behavior and that's why I ended up asking here. But it's good to raise it, probably we should at least mention this in ACD

Anders Elowsson — 16/03/2026, 23:38
Yes, the behavior that @pk910 describes interacts very well with EIP-8184, LUCID encrypted mempool. The reason is that this allows for the recovery mechanism that LUCID relies on to ensure that if you observe conditions allowing you to release a key, we guarantee that the transactions becomes canonical. Specifically, we wish to wait with adding the beacon block root until after the decrypted transactions have executed. It would thus be ideal if you could add all the missing beacon block roots at the same time when executing the first full payload, as this would simplify things greatly from a LUCID perspective.

pk910 — 16/03/2026, 23:42
Adding multiple entries to the ring buffer is unfortunately not possible with the current system contract architecture :/
So we either have to do dirty storage hacks to bypass the on chain insertion logic, or deploy a new beacon roots system contract. 
^cc @lightclient  (probably has more insights into the system contracts) 

lightclient — Yesterday at 00:24
wdym by adding multiple entries to the ring buffer? if you want to only publish the beacon block hash / execution block hash after decryption, it seems like they can be done sequentially. you just have to work backwards to figure out what time stamp the evm was supposed to be at that time and submit it with that evm context
but that might have other downstream implications on consumers who expect the hashes available at n+1

pk910 — Yesterday at 03:14
Ah yea, that's another approach, just executing the missed beacon root updates in their own execution context "in between" blocks..
I wonder if that has any implications with the zk world, as these updates are not part of the block execution context itself?

Anders Elowsson — Yesterday at 03:17
What does "in between" mean? I ask because for LUCID it would be best to execute all missed beacon root updates sequentially during processing of the first full block after a string of empty ones, so wanna check if that is the idea there..

pk910 — Yesterday at 03:23
yea, I think that matches the idea.

assume this:
slots:   B0 B1 B2 B3 B4
payload: E0 E1 -- -- E4

block B2 & B3 are empty. 
For B4 the el clients have to execute the beacon root updates for E2 + E3 in their own execution context (with the block time of B2 & B3), then execute E4. all in one go, if the block is invalid, revert all. 

We'll need a change in the engine api to pass a list of empty block times + roots 
And we should probably delay this change till we have a merged glamsterdam network. Feels like a EL heavy change, that'll cause conflicts when merging the bal branches

potuz — Yesterday at 04:07
It seems to me the current behavior is probably good. There's probably only very niche applications in having those blockroots in the evm (notice that just the child root is good enough to get the others in the CL and prove against them)

Anders Elowsson — Yesterday at 04:16
That's an interesting point. I'll let you guys work it out 🙂

pk910 — Yesterday at 04:23
It'd definitely simplify things when leaving it as is.  But the behavior feels a bit inconsistent then :/
In the example above, we'd never have the block roots of B1 & B2 on chain. the lack of B1 feels problematic as it's a full block with all kind of operations.
sure, you can go back the path from B3 to B1 via the ParentBlockRoots in proofs. But that path could get quite long under bad network conditions and several empty blocks in a row?

potuz — Yesterday at 04:30
They are on-chain, they're not in that system contract but they are on-chain. The main purpose of that contract is IMO to be able to prove against the CL as soon as you can. You cannot prove against B1 and B2 at all while there aren't payloads being executed. And you can as soon as you get E4 including B3 because the state for B3 also has commitments to B1 and B2.
So from the point of view of the user here, I very much doubt it makes a lot of difference.
I can imagine that proving that something happened in a particular beacon block root becomes much more difficult though, cause instead of proving directly against one of the blockroots available directly in the EVM you need to provide a merkle proof against the latest one. But this only affects proofs not about a current status (like being slashed or this or that) but against having received this CL reward at this specific block for example

pk910 — Yesterday at 04:44
Oh yea,  you're right.  The state has a list of all recent block roots anyway, so the proof path length keeps the same.
It's only getting a bit more complex due to proofing through the state,  but guess that's fine.

potuz — Yesterday at 04:58
We should still ping some probable consumers of this.. cc @dgusakov
Don't know anyone at RP here

dgusakov — Yesterday at 14:02
We do prove exact events in exact blocks. Withdrawals, for example. If you are about to introduce the change where some blocks can be obtained directly while others via N jumps to parent, it will be problematic for us for sure

pk910 — Yesterday at 15:30
that's basically exactly what we'Re talking about 😅 
with epbs we won't have every block root on chain anymore.
there are situations where a block root of a regular block does not appear in the beacon roots contract.

the workaround we've discussed above is, that the proof could be extended to not start at the block root that includes the operation that should be proved. Instead, extend the proof to go from the last available block_root -> state_root -> block_roots[]   and from there to the individual operation (eg. withdrawals).

it'll make the proofs longer and more complex.
proving withdrawals could actually be problematic with epbs 🤔 
we don't have a execution_payload_envelope_root or execution_payload_root in the bid,   so there's no merkle proof path towards the ExecutionPayload?
cc @potuz 

Adrian Sutton — Yesterday at 15:36
oof, that sounds painful.  Is there a way to know if a block hash is or isn't going to be available easily? OP Stack uses the on-chain block hash history as part of proving derivation as it makes it way faster to jump a long way back in history without having to walk through the block chain individually. We can be flexible about the process and already handle doing the lookup in multiple hops but would have to make adjustments if we can't look up an arbitrary block within the history range.

pk910 — Yesterday at 15:38
parent roots of blocks with empty payloads won't be on chain.   but don't mix the block hash with beacon block roots.
we're talking about the EIP-4788 contract,  not EIP-2935 (block hashes) 

Adrian Sutton — Yesterday at 15:41
Ah, that's fine then - I only care about EL block hashes in this case.  OP Stack passes through the last beacon block header from L1 but we'll just continue to mirror whatever that is on L1 so should be fine. 

potuz — Yesterday at 15:47
Withdrawals that were deducted in block N on the CL have to be paid in block N+k in the EL with some k>=0. The block root that will appear in the contract is N+k-1 but most importantly, they are cashed on the state until they are paid. If withdrawals are proved against inclusion in a Payload I believe it shouldn't be much different. If they are proved against deduction in the CL at a particular CL block yes this can be an issue, but if it's proven against the beacon state of that block (that cashed the withdrawals) then it should be fine as well

pk910 — Yesterday at 16:01
does it make sense to add a execution_payload_root to the bid? that way apps can construct merkle proofs into the payload too.
could be interesting if we go forward with ssz and transactions & co become structured ssz objects too.  then everything within a block or its payload would be provable via the block root

potuz — Yesterday at 16:12
That can't be added
Because there are fields in the envelope that can't be known at that time. Incidentally withdrawals is one of them

potuz — Yesterday at 16:40
But yes, life for these kinds of problems would be much easier if EL devs bit the bullet and moved to SSZ in the whole pipeline.

pk910 — Yesterday at 16:45
I don't fully get that argument.   the block_hash is included in a bid, which is the hash over the whole execution block header, which includes the withdrawals_root.  So there is already a cryptographic commitment for the included withdrawals in the bid?
the execution_payload_root would just be another representation of the same data?

Anyway, I'll gonna dig into the proof complexity a bit.. let's see how big these proofs really get when going through the state / checking expected_withdrawals.

potuz — Yesterday at 16:50
The bid does not include the beacon block root of the current root, it includes the beacon block root of the parent beacon block. In the happy case the bid is for a payload that will fullfil withdrawals for the current beacon block which is not known yet at the time of bid submission
The builder can always include a HTR for the full execution payload object within the envelope, but it cannot provide an HTR of the full envelope where some extra necessary data is included.

pk910 — Yesterday at 16:52
that's what I'm talking about 😉  I've never mentioned a execution_payload_envelope_root

potuz — Yesterday at 17:01
ah yeah, for just the payload's HTR would work for this particular case, and yes, getting the payload HTR instead of the hash in the bid would be WAAAAY better, now go an advocate in ACDE for them to include this 😄
BTW, we could do this independent of the EL devs if we agreed, the bid is on the CL only

pk910 — Yesterday at 17:28
so, a proof from a beacon root  to a previous beacon root via block_roots needs 22 sibling hashes.
going from a block root to a specific withdrawal root needs 14 sibling hashes.
total 36 hashes per merkle proof verification for a withdrawal.

right now, proving a specific withdrawal directly from the right beacon root and traversing through body_root -> execution_payload -> withdrawals needs 16 sibling hashes.

so the complexity / proof size and gas costs would basically double when leaving it as is.
if we get the beacon roots contract fixed, we can get rid of the first 22 siblings,  making the proof via the state actually cheaper than before 