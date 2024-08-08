# Abstract

This document specifies a new BIP0157 client filter type that contains the serialized utreexo state for a given block. This message can be used in protocols where clients require the utreexo state for an arbitrary block.

# Table of Contents

1. [License](#license)
2. [Motivation](#motivation)
3. [Specification](#specification)

# License

This BIP is licensed under the BSD 3-clause license.

# Motivation

Some protocols[^1] require the knoweledge of the UTXO set at an arbitrary height. The Bitcoin P2P network currently doesn't give any method to obtain such data, which should be independently calculated by each node by applying all transactions from the genesis block. The main reason to not serve the UTXO set for other peers is the elevated amount of resources spent by nodes to support it, requiring nodes to keep the UTXO set for each block, rather than just the last the latest one. However, utreexo states are small, under a kb, and can be efficiently stored and shared with one's peers. This BIP defines a method for obtaining the UTXO set state for a given block, directly from the P2P network.

# Specification

A new BIP0157 client-side filter UTREEXO_FILTER, with type `0x01`, is defined for retrieving the UTXO set state as a the Utreexo roots for that block. The filter data is defined as follows

## Serialization

| Name | Type | Description |
|------|------|-------------|
| nLeaves | uint64 | How many leaves we have in the accumulator encoded in consensus order |
| hashes |  Array of 32-bytes hashes | The accumulator's roots encoded in ascending order (from the lowest root to the highest, using the consensus serialization of hashes) |

## Signalling

This BIP allocates a new service bit 1 << 13 used to signal support for UTREEXO_FILTER

## Why using BIP0157

Rather than defining a completely new network message, requiring more work for implementers that already have code for indexing and serving BIP0157/BIP0158. We decided to reuse the existing infrastructure and only defining a new filter type.

[^1]: E.g. [proof of work fraud proofs](https://blog.dlsouza.lol/2023/09/28/pow-fraud-proof.html) and [out-of-order-sync](https://blog.bitmex.com/out-of-order-block-validation-with-utreexo-accumulators/)
