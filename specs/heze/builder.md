# Heze -- Honest Builder

*Note*: This document is a work-in-progress for researchers and implementers.

<!-- mdformat-toc start --slug=github --no-anchors --maxlevel=6 --minlevel=2 -->

- [Introduction](#introduction)
- [Builder activities](#builder-activities)
  - [Constructing the `SignedExecutionPayloadBid`](#constructing-the-signedexecutionpayloadbid)

<!-- mdformat-toc end -->

## Introduction

This document represents the changes to be made in the code of an "honest
builder" to implement Heze.

## Builder activities

### Constructing the `SignedExecutionPayloadBid`

*Note*: In addition to setting `bid.inclusion_list_bits`, the builder sets the
JIT and AOT blob KZG commitments.

1. Set `bid.inclusion_list_bits` to
   `get_inclusion_list_bits(get_inclusion_list_store(), state, Slot(bid.slot - 1))`.
2. Set `bid.jit_blob_kzg_commitments` to the commitments for JIT blobs included
   in the execution payload. The builder constructs these from the JIT blobs
   directly available to them.
3. Set `bid.aot_blob_kzg_commitments` to the commitments obtained from
   `AOTDataColumnSidecar`s received for active tickets targeting `bid.slot`.
   The `kzg_commitments` field of any received `AOTDataColumnSidecar` contains
   the full list of AOT blob KZG commitments for the corresponding ticket.