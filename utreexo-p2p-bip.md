# Abstract
Utreexo creates a compact representation of the UTXO set that only takes a couple of kilobytes. When spending a transaction, one must provide an inclusion proof for the UTXOs being spent. This BIP defines the networking-layer changes needed to allow nodes to exchange utreexo proofs. The present document **does not** describe how to validate blocks and transactions using the provided data, check BIP-VALIDATION for more details.

# Table of Contents
1. [License](#license)
2. [Specification](#specification)
    1. [Block proofs](#block-proofs)
        1. [Compact leaf data](#compact-leaf-data)
        2. [Reconstructable Script](#reconstructable-script)
        3. [Udata](#udata)
        4. [`block` message](#block-message)

# License
This BIP is licensed under the BSD 3-clause license.

# Specification

For utreexo P2P usage, no new message types are defined.  The same flow of messages and state machines is used, with modifications to existing network messages, including new INV types.

## Handshake

TODO: need to describe handshake and version bits here.  Also decide what behavior utreexo nodes should have when non-utreexo nodes connect to them.

## Block proofs

This section defines the network message a node can use to learn about proofs and the UTXO data necessary to validate a block. This message is send in response to a `getdata` request where the `inv` type contains the `UTREEXO_BLOCK` inv element. Each field and its serialization are given bellow. 

### Compact leaf data

For a CSN to learn the data associated with a UTXO, it must ask for a peer that has it. To authenticate this data, it is committed into the accumulator, and therefore cannot be changed by peer. The committed data is defined in BIP-VALIDATION#LEAF_DATA, but for some information in the leaf data, the receiving peer might already have it, so sending it again is a waste of bandwidth. To save that bandwidth, we only send a Compact Leaf Data, that contains all missing information for the receiving peer to reconstruct the full leaf data. A compact leaf data is defined as:

|Field | type | Description |
|--------|--------|-----------------|
| header code | 4-bytes little-endian unsigned integer | This is a value obtained by left shifting the block height that confirmed this transaction, and then OR-ing it with 1, only if this transaction is a coinbase. |
amount | 8-bytes little-endian unsigned integer |The amount in sats locked on this output
scriptPubkey |reconstructable scriptPublickey | The scriptPubkey in a reconstructable format, see [Reconstructable Script](#Reconstructable-Script) for more details |

#### Reconstructable Script

For some script types (e.g. `ScriptHash`, `PubkeyHash`, `WitnessScriptHash`, `WitnessPubkeyHash`) the actual locking condition is not in the scriptPubkey, but a hash of it.  The script which is evaluated is provided as an element of the scriptSig or witness data.  Therefore, we can safely just omit the locking script hash from the UTXO data and reconstruct it from the witness or scriptSig. A Reconstructable Script is a tagged union that lets nodes recreate the script without necessarily providing redundant information. If we can reconstruct the committed hash from the transaction data, we just say which type should we expect. Only if the actual script cannot be reconstructed from transaction data, like in the case of taproot outputs, we send the actual script. The serialization and tag values are given below:
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
| Batch Proof | A batch proof over all inputs being spent that haven’t been created in this block (see BIP-VALIDATION for more details) |
| Remember Index | For future developments, should be 0x00 |
| Leaf data | An array of compact leaf data |

### `block` message
| Field | Description |
|--------|-----------------|
| Block data | The usual block data, with or without witness |
| Udata | All the utreexo context needed for this block, as defined in [Udata](Udata) |

| Field | Description |
|--------|-----------------|
| Block data | The usual block data, with or without witness |
| Udata | All the utreexo context needed for this block, as defined in [Udata](Udata) |


For more information about how to use a Utreexo bock and the Udata, referer to BIP-VALIDATION.
