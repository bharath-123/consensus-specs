# Heze -- Data Availability Sampling Core

<!-- mdformat-toc start --slug=github --no-anchors --maxlevel=6 --minlevel=2 -->

- [Introduction](#introduction)
- [Helpers](#helpers)
  - [New `merge_data_column_sidecars`](#new-merge_data_column_sidecars)
- [Column merging](#column-merging)
- [Reconstruction and cross-seeding](#reconstruction-and-cross-seeding)
- [Custody](#custody)
  - [Custody requirement](#custody-requirement)

<!-- mdformat-toc end -->

## Introduction

With the introduction of two-tier blob propagation (AOT and JIT blobs), data
columns for a given block arrive from multiple independent sources during gossip:

- **JIT data columns**: Arrive as `DataColumnSidecar`s on the
  `data_column_sidecar_{subnet_id}` subnets. Each sidecar for column index `i`
  carries one cell per JIT blob at that column index.
- **AOT data columns (per ticket)**: Arrive as `AOTDataColumnSidecar`s on the
  `aot_data_column_sidecar_{subnet_id}` subnets. Each AOT ticket holder sends
  their own set of sidecars. Each sidecar for column index `i` carries one cell
  per AOT blob **for that ticket** at that column index. A given block may
  reference multiple AOT tickets, each with its own blob set and KZG
  commitments.

The blobs for a given block do not all arrive in the same column sidecar --
they are spread across JIT and AOT sidecars, each carrying a single cell per
blob for its column index.

Once a block is known and the JIT and AOT columns for a given column index are
available, they are **merged** into a single `DataColumnSidecar` per column
index. This merged form is what nodes store, custody, and serve via the
`DataColumnsByRoot` and `DataColumnsByRange` Req/Resp methods. By merging at
inclusion time, checkpoint sync, backfill, and range sync remain unchanged from
Gloas.

*Note*: The `DataColumnSidecar` container is unchanged from Gloas. Its column
list remains bounded by `MAX_BLOB_COMMITMENTS_PER_BLOCK`, which accommodates
the combined JIT and AOT blob cells.

## Helpers

### New `merge_data_column_sidecars`

*[New in Heze:EIP-XXXX]*

```python
def merge_data_column_sidecars(
    jit_sidecar: DataColumnSidecar,
    aot_sidecars: Sequence[AOTDataColumnSidecar],
) -> DataColumnSidecar:
    """
    Merge a JIT data column sidecar and zero or more AOT data column sidecars
    for the same column index into a single ``DataColumnSidecar``.

    The merged column contains JIT blob cells first, followed by AOT blob cells
    in the order the corresponding tickets appear in the bid's
    ``aot_blob_kzg_commitments``. This ordering ensures the merged column cells
    align with the concatenation of ``bid.jit_blob_kzg_commitments`` and
    ``bid.aot_blob_kzg_commitments``.

    All sidecars MUST have the same column index.
    """
    assert all(
        aot_sidecar.index == jit_sidecar.index for aot_sidecar in aot_sidecars
    )

    merged_column = list(jit_sidecar.column)
    merged_kzg_proofs = list(jit_sidecar.kzg_proofs)
    for aot_sidecar in aot_sidecars:
        merged_column.extend(aot_sidecar.column)
        merged_kzg_proofs.extend(aot_sidecar.kzg_proofs)

    return DataColumnSidecar(
        index=jit_sidecar.index,
        column=merged_column,
        kzg_proofs=merged_kzg_proofs,
        slot=jit_sidecar.slot,
        beacon_block_root=jit_sidecar.beacon_block_root,
    )
```

## Column merging

When a node receives an `ExecutionPayloadEnvelope` and the corresponding block
is known, the node SHOULD merge the JIT and AOT data columns for each column
index into a single `DataColumnSidecar`:

1. For each column index `i` in the node's custody:
   - Take the JIT `DataColumnSidecar` for column `i` (received via
     `data_column_sidecar_{subnet_id}` gossip).
   - Take the `AOTDataColumnSidecar`s for column `i` from each ticket referenced
     by the bid's `aot_blob_kzg_commitments`, ordered to match the bid's
     commitment ordering.
   - Call `merge_data_column_sidecars(jit_sidecar, aot_sidecars)` to produce the
     merged `DataColumnSidecar` for column `i`.
2. Store the merged `DataColumnSidecar`s for custody and serving.

The merged `DataColumnSidecar` can be verified using the existing
`verify_data_column_sidecar` and `verify_data_column_sidecar_kzg_proofs`
helpers with the combined KZG commitments
(`bid.jit_blob_kzg_commitments + bid.aot_blob_kzg_commitments`).

## Reconstruction and cross-seeding

Reconstruction only applies during forward sync (real-time gossip), where JIT
and AOT columns arrive separately and a node may need to recover missing columns
before merging. Reconstruction is performed independently for each blob source
since they have different KZG commitments and blob counts:

- **JIT columns**: If a node obtains 50%+ of all JIT columns for a given block,
  it SHOULD reconstruct the full JIT matrix via `recover_matrix` with
  `blob_count = len(bid.jit_blob_kzg_commitments)`. The partial matrix MUST
  contain only cells from JIT `DataColumnSidecar`s.
- **AOT columns (per ticket)**: For each ticket, if a node obtains 50%+ of that
  ticket's columns, it SHOULD reconstruct using `recover_matrix` with
  `blob_count = ticket.blob_count`. The partial matrix MUST contain only cells
  from `AOTDataColumnSidecar`s belonging to that same `ticket_id`. Cells from
  different tickets MUST NOT be mixed.

Once reconstruction completes, the recovered columns are merged as described
above and stored as merged `DataColumnSidecar`s.

Reconstructed JIT columns MAY be exposed to the
`data_column_sidecar_{subnet_id}` subnet if subscribed. Reconstructed AOT
columns MAY be broadcast to the `aot_data_column_sidecar_{subnet_id}` subnet
by constructing a new `AOTDataColumnSidecar` with the recovered cells/proofs
and reusing the `ticket_id`, `target_slot`, and `blob_info_signature` from any
existing sidecar for the same ticket. This is valid because the signature is
over the `AOTBlobInfo` (not the individual sidecar).

Nodes MAY delay reconstruction to allow time for additional columns to arrive
over the network. If delaying, nodes may use a random delay to desynchronize
reconstruction and reduce CPU load.

## Custody

### Custody requirement

Custody requirements are unchanged from Gloas. Nodes custody merged
`DataColumnSidecar`s for their assigned custody groups. When syncing, a node
MUST backfill merged `DataColumnSidecar`s from all of its custody groups using
the existing `DataColumnsByRoot` and `DataColumnsByRange` Req/Resp methods.
Since the merged form is a standard `DataColumnSidecar`, checkpoint sync,
backfill, and range sync require no protocol changes.
