# Heze -- Networking

*Note*: This document is a work-in-progress for researchers and implementers.

<!-- mdformat-toc start --slug=github --no-anchors --maxlevel=6 --minlevel=2 -->

- [Introduction](#introduction)
- [Modifications in Heze](#modifications-in-heze)
  - [Configuration](#configuration)
  - [Containers](#containers)
    - [New `AOTDataColumnSidecar`](#new-aotdatacolumnsidecar)
  - [Helpers](#helpers)
    - [Modified `compute_fork_version`](#modified-compute_fork_version)
    - [New `verify_aot_data_column_sidecar`](#new-verify_aot_data_column_sidecar)
    - [New `verify_aot_data_column_sidecar_kzg_proofs`](#new-verify_aot_data_column_sidecar_kzg_proofs)
    - [New `aot_data_column_sidecar_to_data_column_sidecar`](#new-aot_data_column_sidecar_to_data_column_sidecar)
  - [The gossip domain: gossipsub](#the-gossip-domain-gossipsub)
    - [Topics and messages](#topics-and-messages)
      - [Global topics](#global-topics)
        - [`inclusion_list`](#inclusion_list)
      - [Blob subnets](#blob-subnets)
        - [`data_column_sidecar_{subnet_id}`](#data_column_sidecar_subnet_id)
        - [`aot_data_column_sidecar_{subnet_id}`](#aot_data_column_sidecar_subnet_id)
  - [The Req/Resp domain](#the-reqresp-domain)
    - [Messages](#messages)
      - [InclusionListByCommitteeIndices v1](#inclusionlistbycommitteeindices-v1)

<!-- mdformat-toc end -->

## Introduction

This document contains the consensus-layer networking specifications for Heze.

The specification of these changes continues in the same format as the network
specifications of previous upgrades, and assumes them as pre-requisite.

## Modifications in Heze

### Configuration

| Name                           | Value             | Description                                                |
| ------------------------------ | ----------------- | ---------------------------------------------------------- |
| `MAX_REQUEST_INCLUSION_LIST`   | `2**4` (= 16)     | Maximum number of inclusion list in a single request       |
| `MAX_BYTES_PER_INCLUSION_LIST` | `2**13` (= 8,192) | Maximum size of the inclusion list's transactions in bytes |

### Containers

#### New `AOTDataColumnSidecar`

*[New in Heze:EIP-XXXX]*

`AOTDataColumnSidecar` carries a single data column for an AOT blob submission.
It includes the `ticket_id` and `target_slot` to identify the corresponding
ticket, and the BLS signature of the original `AOTBlobInfo` for
authentication.

*Note*: The signature here is the signature over the original `AOTBlobInfo`,
not over each individual `AOTDataColumnSidecar`.

```python
class AOTDataColumnSidecar(Container):
    index: ColumnIndex
    column: List[Cell, MAX_AOT_BLOB_COMMITMENTS_PER_BLOCK]
    kzg_commitments: List[KZGCommitment, MAX_AOT_BLOB_COMMITMENTS_PER_BLOCK]
    kzg_proofs: List[KZGProof, MAX_AOT_BLOB_COMMITMENTS_PER_BLOCK]
    # [New in Heze:EIP-XXXX]
    ticket_id: TicketId
    target_slot: Slot
    # BLS signature of the original AOTBlobInfo (signed by the ticket's registered pubkey)
    blob_info_signature: BLSSignature
```

### Helpers

#### Modified `compute_fork_version`

```python
def compute_fork_version(epoch: Epoch) -> Version:
    """
    Return the fork version at the given ``epoch``.
    """
    if epoch >= HEZE_FORK_EPOCH:
        return HEZE_FORK_VERSION
    if epoch >= GLOAS_FORK_EPOCH:
        return GLOAS_FORK_VERSION
    if epoch >= FULU_FORK_EPOCH:
        return FULU_FORK_VERSION
    if epoch >= ELECTRA_FORK_EPOCH:
        return ELECTRA_FORK_VERSION
    if epoch >= DENEB_FORK_EPOCH:
        return DENEB_FORK_VERSION
    if epoch >= CAPELLA_FORK_EPOCH:
        return CAPELLA_FORK_VERSION
    if epoch >= BELLATRIX_FORK_EPOCH:
        return BELLATRIX_FORK_VERSION
    if epoch >= ALTAIR_FORK_EPOCH:
        return ALTAIR_FORK_VERSION
    return GENESIS_FORK_VERSION
```

#### New `verify_aot_data_column_sidecar`

*[New in Heze:EIP-XXXX]*

```python
def verify_aot_data_column_sidecar(
    sidecar: AOTDataColumnSidecar,
    kzg_commitments: List[KZGCommitment, MAX_AOT_BLOB_COMMITMENTS_PER_BLOCK],
) -> bool:
    """
    Verify if the AOT data column sidecar is structurally valid.
    """
    # The sidecar index must be within the valid range
    if sidecar.index >= NUMBER_OF_COLUMNS:
        return False

    # A sidecar for zero blobs is invalid
    if len(sidecar.column) == 0:
        return False

    # The column length must be equal to the number of commitments/proofs
    if len(sidecar.column) != len(kzg_commitments) or len(sidecar.column) != len(
        sidecar.kzg_proofs
    ):
        return False

    return True
```

#### New `verify_aot_data_column_sidecar_kzg_proofs`

*[New in Heze:EIP-XXXX]*

```python
def verify_aot_data_column_sidecar_kzg_proofs(
    sidecar: AOTDataColumnSidecar,
    kzg_commitments: List[KZGCommitment, MAX_AOT_BLOB_COMMITMENTS_PER_BLOCK],
) -> bool:
    """
    Verify if the KZG proofs for an AOT data column sidecar are correct.
    """
    # The column index also represents the cell index
    cell_indices = [CellIndex(sidecar.index)] * len(sidecar.column)

    # Batch verify that the cells match the corresponding commitments and proofs
    return verify_cell_kzg_proof_batch(
        commitments_bytes=kzg_commitments,
        cell_indices=cell_indices,
        cells=sidecar.column,
        proofs_bytes=sidecar.kzg_proofs,
    )
```

#### New `aot_data_column_sidecar_to_data_column_sidecar`

*[New in Heze:EIP-XXXX]*

```python
def aot_data_column_sidecar_to_data_column_sidecar(
    sidecar: AOTDataColumnSidecar,
    beacon_block_root: Root,
    slot: Slot,
) -> DataColumnSidecar:
    return DataColumnSidecar(
        index=sidecar.index,
        column=sidecar.column,
        kzg_proofs=sidecar.kzg_proofs,
        slot=slot,
        beacon_block_root=beacon_block_root,
    )
```

### The gossip domain: gossipsub

#### Topics and messages

The new topics along with the type of the `data` field of a gossipsub message
are given in this table:

| Name             | Message Type          |
| ---------------- | --------------------- |
| `inclusion_list` | `SignedInclusionList` |

The following subnet topics are added in Heze:

| Name                                   | Message Type           |
| -------------------------------------- | ---------------------- |
| `aot_data_column_sidecar_{subnet_id}` | `AOTDataColumnSidecar` |

##### Global topics

Heze introduces a new global topic for inclusion lists.

###### `inclusion_list`

This topic is used to propagate signed inclusion list as `SignedInclusionList`.
The following validations MUST pass before forwarding the `inclusion_list` on
the network, assuming the alias `message = signed_inclusion_list.message`:

- _[REJECT]_ The size of `message.transactions` is within upperbound
  `MAX_BYTES_PER_INCLUSION_LIST`.
- _[REJECT]_ The slot `message.slot` is equal to the previous or current slot.
- _[IGNORE]_ The slot `message.slot` is equal to the current slot, or it is
  equal to the previous slot and the current time is less than
  `get_attestation_due_ms(epoch)` milliseconds into the slot.
- _[IGNORE]_ The `inclusion_list_committee` for slot `message.slot` on the
  current branch corresponds to `message.inclusion_list_committee_root`, as
  determined by
  `hash_tree_root(inclusion_list_committee) == message.inclusion_list_committee_root`.
- _[REJECT]_ The validator index `message.validator_index` is within the
  `inclusion_list_committee` corresponding to
  `message.inclusion_list_committee_root`.
- _[IGNORE]_ The `message` is either the first or second valid message received
  from the validator with index `message.validator_index`.
- _[REJECT]_ The signature of `signed_inclusion_list.signature` is valid with
  respect to the validator's public key.

##### Blob subnets

###### `data_column_sidecar_{subnet_id}`

*[Modified in Heze:EIP-XXXX]*

The validation rules are modified to use `jit_blob_kzg_commitments` instead of
`blob_kzg_commitments`. The following validations MUST pass before forwarding
the `sidecar: DataColumnSidecar` on the network, assuming the alias
`bid = block.body.signed_execution_payload_bid.message` where `block` is the
`BeaconBlock` associated with `sidecar.beacon_block_root`:

- _[IGNORE]_ A valid block for the sidecar's `slot` has been seen (via gossip or
  non-gossip sources). If not yet seen, a client MUST queue the sidecar for
  deferred validation and possible processing once the block is received or
  retrieved.
- _[REJECT]_ The sidecar's `slot` matches the slot of the block with root
  `beacon_block_root`.
- _[REJECT]_ The sidecar is valid as verified by
  `verify_data_column_sidecar(sidecar, bid.jit_blob_kzg_commitments)`.
- _[REJECT]_ The sidecar is for the correct subnet -- i.e.
  `compute_subnet_for_data_column_sidecar(sidecar.index) == subnet_id`.
- _[REJECT]_ The sidecar's column data is valid as verified by
  `verify_data_column_sidecar_kzg_proofs(sidecar, bid.jit_blob_kzg_commitments)`.
- _[IGNORE]_ The sidecar is the first sidecar for the tuple
  `(sidecar.beacon_block_root, sidecar.index)` with valid kzg proof.

###### `aot_data_column_sidecar_{subnet_id}`

*[New in Heze:EIP-XXXX]*

This topic is used to propagate AOT data column sidecars ahead of block time.
AOT data columns are pre-propagated by ticket holders who have purchased blob
tickets on the execution layer. The subnet assignment follows the same mapping
as JIT data columns:
`compute_subnet_for_data_column_sidecar(sidecar.index) == subnet_id`.

*Note*: Clients obtain the set of active tickets (including `ticket_id`,
`blob_count`, `bls_pubkey`, and `target_slot`) for the current and upcoming
slots via the `engine_forkchoiceUpdated` API response. This ticket information
is required to validate incoming AOT data column sidecars.

The following validations MUST pass before forwarding the
`sidecar: AOTDataColumnSidecar` on the network. Let `ticket` be the ticket
corresponding to `sidecar.ticket_id` as obtained from the execution layer via
the engine API:

- _[IGNORE]_ The sidecar's `target_slot` is within the propagation window --
  i.e.
  `current_slot <= sidecar.target_slot <= current_slot + AOT_PROPAGATION_WINDOW_SLOTS`.
- _[REJECT]_ The `ticket_id` corresponds to a valid, active ticket known to the
  node -- i.e. the ticket exists in the set of active tickets returned by the
  engine API.
- _[REJECT]_ The sidecar's `target_slot` matches the ticket's target slot --
  i.e. `sidecar.target_slot == ticket.target_slot`.
- _[REJECT]_ The `blob_info_signature` is a valid BLS signature over the
  reconstructed `AOTBlobInfo` with respect to the ticket's registered BLS
  public key -- i.e.
  `is_valid_aot_blob_info_signature(SignedAOTBlobInfo(message=aot_blob_info, signature=sidecar.blob_info_signature), ticket.bls_pubkey)`
  returns `True`, where `aot_blob_info` is reconstructed from the sidecar's
  KZG commitments.
- _[REJECT]_ The sidecar is structurally valid as verified by
  `verify_aot_data_column_sidecar(sidecar, aot_blob_info.blob_kzg_commitments)`.
- _[REJECT]_ The sidecar is for the correct subnet -- i.e.
  `compute_subnet_for_data_column_sidecar(sidecar.index) == subnet_id`.
- _[REJECT]_ The sidecar's column data is valid as verified by
  `verify_aot_data_column_sidecar_kzg_proofs(sidecar, aot_blob_info.blob_kzg_commitments)`.
- _[IGNORE]_ The sidecar is the first valid sidecar for the tuple
  `(sidecar.ticket_id, sidecar.index)` with valid KZG proof.

### The Req/Resp domain

#### Messages

##### InclusionListByCommitteeIndices v1

**Protocol ID:** `/eth2/beacon_chain/req/inclusion_list_by_committee_indices/1/`

For each successful `response_chunk`, the `ForkDigest` context epoch is
determined by `compute_epoch_at_slot(signed_inclusion_list.message.slot)`.

Per `fork_version = compute_fork_version(epoch)`:

<!-- eth_consensus_specs: skip -->

| `fork_version`      | Chunk SSZ type             |
| ------------------- | -------------------------- |
| `HEZE_FORK_VERSION` | `heze.SignedInclusionList` |

Request Content:

```
(
  slot: Slot
  committee_indices: Bitvector[INCLUSION_LIST_COMMITTEE_SIZE]
)
```

Response Content:

```
(
  List[SignedInclusionList, MAX_REQUEST_INCLUSION_LIST]
)
```
