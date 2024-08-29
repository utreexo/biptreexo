# Table of Contents
1. [License](#license)
2. [Specification](#specification)
    1. [Block proofs](#block-proofs)
        1. [Compact leaf data](#compact-leaf-data)
        2. [Reconstructable Script](#reconstructable-script)
        3. [Udata](#udata)
        4. [`block` message](#block-message)

# Abstract
This BIP defines the networking-layer changes needed to allow nodes to exchange utreexo proofs. The present document **does not** describe how to validate blocks and transactions using the provided data, check BIP-VALIDATION for more details.

# Motivation
Utreexo creates a compact representation of the UTXO set that is takes less than a kilobyte. However, when spending a transaction, one must provide the preimages of the committed hashes in the accumulator along with the inclusion proof of the committed hash of the UTXOs spent. New mechanisms are defined for the additional data to be propagated.

# License
This BIP is licensed under the BSD 3-clause license.

# Specification

For utreexo P2P usage, no new message types are defined. The same flow of messages and state machines is used, with modifications to existing network messages, including new INV types.

## Handshake

TODO: need to describe handshake and version bits here.  Also decide what behavior utreexo nodes should have when non-utreexo nodes connect to them.

## Block proofs

This section defines the network message a node can use to learn about proofs and the UTXO data necessary to validate a block. This message is send in response to a `getdata` request where the `inv` type contains the `UTREEXO_BLOCK` inv element. Each field and its serialization are given bellow. 

### Compact leaf data

A Compact State Node (CSN) is a node that reprsents the UTXO set as the accumulator state. For a CSN to learn the data associated with a UTXO, it must ask for a peer that has it. To authenticate this data, it is committed into the accumulator, and therefore cannot be changed by peer. The committed data is defined in BIP-VALIDATION, but for some information in the leaf data, the receiving peer might already have it, so sending it again is a waste of bandwidth. To save that bandwidth, we only send a Compact Leaf Data, that contains all missing information for the receiving peer to reconstruct the full leaf data. A compact leaf data is defined as:

|Field | type | Description |
|--------|--------|-----------------|
| header code | 4-bytes little-endian unsigned integer | This is a value obtained by left shifting the block height that confirmed this transaction, and then OR-ing it with 1, only if this transaction is a coinbase.
| amount | 8-bytes little-endian unsigned integer |The amount in sats locked on this output
| scriptPubkey | reconstructable scriptPublickey | The scriptPubkey in a reconstructable format, see [Reconstructable Script](#Reconstructable-Script) for more details |

#### Reconstructable Script

For some script types (e.g. `ScriptHash`, `PubkeyHash`, `WitnessScriptHash`, `WitnessPubkeyHash`) the actual locking condition is not in the scriptPubkey, but a hash of it. The script which is evaluated is provided as an element of the scriptSig or witness data. Therefore, we can safely just omit the locking script hash from the UTXO data and reconstruct it from the witness or scriptSig. A Reconstructable Script is a tagged union that lets nodes recreate the script without necessarily providing redundant information. If we can reconstruct the committed hash from the transaction data, we just say which type should we expect. Only if the actual script cannot be reconstructed from transaction data, like in the case of taproot outputs, we send the actual script. The serialization and tag values are given below:

Reconstructable script
| Field | Type | Description | Required |
|---------|--------|-----------------|--------------|
| tag     | 1-byte unsigend integer | What kind of script is this | yes |
| length | varint | The script length | only if tag type is 0x00 |
| script  | variable-length slice | The actual script |  only if tag type is 0x00 |

The possible values for the tag are:

| Value | Script Type |
|---------|-----------------|
| 0x00 |  Other         |
| 0x01 |  Pubkey Hash |
| 0x02 |  WitnessV0PubkeyHash |
| 0x03 |  ScriptHash |
| 0x04 | WitnessV0ScriptHash |

### Udata

Udata is all the data needed to validate that block, along with some extra hints for caching.

| Field | Descrition |
|--------|---------------|
| Batch Proof | A batch proof over all inputs being spent that haven’t been created in this block (see BIP-VALIDATION for more detail) |
| Remember Index | Reserved for future caching. Should be 0x00 |
| Leaf data | An array of compact leaf data |

### `block` message

For propagating block messages, new Utreexo Inv Flag is defined at (1<<24) and it is set on MSG_BLOCK and MSG_WITNESS_BLOCK inventory vectors. Inv UTREEXO_BLOCK is 16777218 and Inv WITNESS_UTREEXO_BLOCK is defined at 1090519042.

For block messages that are propagated to Utreexo peers, the encoding follows the below UtreexoEncoding.

| Field | Description |
|--------|-----------------|
| Block data | The usual block data, with or without witness |
| Udata | All the utreexo context needed for this block, as defined in [Udata](Udata) |

For more information about how to use a Utreexo block and the Udata, refer to BIP-VALIDATION.

### `tx` message

#### Relaying invs to Utreexo peers

For transaction propagation, the protocol takes into account that the node may already have proofs cached in the mempool. Receiving the full proof like with the block is wasteful and the protocol for receiving the transaction proof takes this into account.

We introduce a new inventory vector type `MSG_PROOF_HASH` and these inventory vectors are used to communicate the positions of the spent TXOs in the given transaction.
Each position is represented as a uint64 and 4 of them are serialized in little endian format into the 32 byte array. If there are less than 4 to serialize into the 32 byte array, the trailing unused positions are padded with 18446744073709551615, the maximum value that a uint64 value can represent.
For example, when relaying transaction A that spends UTXOs at positions 5 and 10, the Inv message will look like so:

|       Inv      |                                Data                                |
|:--------------:|:------------------------------------------------------------------:|
| MSG_UTREEXO_TX | TXIDA                                                              |
| MSG_PROOF_HASH | 0x0000000000000005000000000000000AFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF |

#### Relaying getdata for transactions to Utreexo peers

When a Utreexo peer receives a inv message for a transaction, it uses the position information received with the inventory message to determine which hashes at which positions are required to verify that the inputs of the transaction exist.
These positions are also serialized into the 32 byte array and uses the same inventory type as the inv message. For example, if the node requests for transaction A and positions 6 and 11 for the proof, it'll look like so:

|       Inv      |                                Data                                |
|:--------------:|:------------------------------------------------------------------:|
| MSG_UTREEXO_TX | TXIDA                                                              |
| MSG_PROOF_HASH | 0x0000000000000006000000000000000BFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF |
