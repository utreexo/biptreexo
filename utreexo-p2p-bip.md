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

# Definitions

Block Headers: The 80 byte block headers we all know and love.  They are tied to a block's transactions via the merkle root field, and the hash of the header is the unique identifier for the entire block.

Block Summaries: Tied to a specific block by including a block hash.  A short (a few kilobytes) message about a block describing which UTXOs are spent in the block.

Block Proof: Also tied to a specific block by including a block hash.  A message containing hash based proof data which proves the validity of UTXOs being spent in the block.  Similar in size to a full block.

# Synchronization

In the simplest setup, there's no difference in how a node downloads the 2nd block and the 2nd block from the current tip.  A syncing node would download the block, which includes the 80 byte header and all transactions, and the full block proof, which provides the UTXO data for all inputs in the block, and proves the inclusion of those UTXOs in the accumulator.  

[note: write this up as a post-IBD flow, should describe that in detail too] 

# IBD optimizations

While using this method a node can securely synchronize, there are several optimizations that can be done for the initial block download (IBD).  These optimizations allow nodes to reduce the total data downloaded during IBD while still getting the same security assurances about the end state of the UTXO set.  

The optimizations require some data to be hard-coded into the program (or supplied with the program in a settings file, command line argument, etc).  Similar to the assumeValid hashes, they give a blockhash that if encountered, causes different behavior, but does not *require* that blockhash to be present.

The hard-coded data consists of two hashes: a Summary Tree Root set and a ?? rootset.  Both of these are tied to a block hash and height.  For example

{"blockhash": 00000000000000000001ed379d0396bf1ac2f8cfd8b40348f08ba7a4990c7b9a, "height": 868352,
"Summary forest roots": [a408dd7455d52009e73167b4a1139d8b473461d53f20ace59867dea0156db6d6, f9a38bd7fabf3b790dd411900553b936fb902cb3760d4c099d74ef49ba6c702d, 3976b5a8d5d46634186c0eb6a043617a3e6f5809a5387d7602e950e08ad1e13b, 3a244de65dd60bc113b43272534693085fc495cf89574d36ad133919d60491b5], 
"Linkup forest roots ": [ef9e0a42da5448deca12051f9074de7d99797c51db6d0bda322943154893b64a, 84672a79b529fdf5a4c09fd59e22fc978834068d4711cbcd65c88df106be7869, 0ed0b17bea326d8d9f866ddb3b35d1e002027dcef8b7035a3b6cc37f0a035b4f, 359b276cf0373964d8a2e99347b727de8eb2a2cc0e029ba5421616d258d55d6d]
}



This is the block hash of block 868352 on mainnet, the root of the Summary Tree, and the Root of the Accumulator Roots Tree.  If block ...7b9a is encountered at height 868352 during the headers-first phase of IBD, the two roots are activated and can be used.




# root root is not static, use the same utreexo algo and append; clients may get extra long proofs but should be OK, !!check!!

## Block Summary

A block summary has the following fields:
numOuts, numIns, [UTXOnumbers]

numOuts is the number of (filtered) outputs in the block, numIns is the number of (filtered) inputs in the block, and UTXONumbers are the canonical numbers for every UTXO spent in the block.

### input / output filtering 
(is this defined anywhere else?  If not, define here -- )

By filtered, there are two ways an output can be dropped from these lists, one of which also applies to inputs.  If an output is clearly not spendable, (eg an OP_RETURN output), it is not included as an output for the count of numOuts.  Also, if an output is created and spent in the same block, it is also skipped for the purposes of the block summary and Utreexo in general; UTXOs like this "never hit the disk" as they are never confirmed, and don't affect the accumulator.

## Summary forest

The summary tree is a merkle forest which contains to Block Summaries for every block.  Archive nodes provide these Block Summaries, along with proofs, to Utreexo nodes performing IBD.  Archive Utreexo nodes keep the entire forest of summaries.  The forest itself only grows at about 3MB per year, but the summaries themselves are significantly larger, up to tens of kilobytes for each block, leading to sizes of around 1GB per year.  The summary data is highly compressable, since it is almost completely runs of correlated / nearby integers.  Archive nodes can compress this data, to save on their disk storage, and it may make sense to also support sending the data in compressed format as well.


## IBD Linkup

Because the size of the state needed to validate blocks is so small with Utreexo, nodes can perform IBD in parallel and out of order.  For example, a computer could divide the task of validating 800,000 blocks into 100 tasks of 8,000 blocks each: blocks 1 through 800, 800 through 1600, 1600 through 2400, and so on.
In order start the 1600 through 2400 IBD task, however, the node should know what the state of the utxo set is at block 1600, so that it can validate and modify the accumulator.  In order to do this, the binary can provide "linkup hints", where the state of the accumulator is given for a desired block hash.  While giving the state of the system might seem at first glance to be introducing a trust assumption, these are not trusted states; the node performing IBD tries out the state given for a block height, but checks that when that state is reached from the thread "below" that it properly links up, with the accumulator state arrived at through full validation matching the state given.  If that link up does not successfully happen, the IBD process should halt; these hints are statements of fact that are hard-coded into the program itself, and if they are false all bets are off about the program.

The format of the Linkup hints is:

Blockhash, Numleaves, [Roots]

Archive nodes create a forest of Linkup hints, so that they can prove, with respect to the Linkup forest roots in a node performing IBD, what their binary has claimed the utxo accumulator state to be at any block height.


(write about proofs working even if they're too long)






# IBD flow





# Specification






IBD message flow:


Normal (same as core) get 2000 headers at a time

-> GetHeaders (x2000)
<- Headers (x2000)


Summary root & proof.

Root of summary

both multi
and their own messages

-> GetBlockSummary (100)
<- BlockSummary (100)

New message types:



post summary, pre tip

get 1 summary, get 1 block, go to next




request proof:  request indexes, server does not deserialize but can truncate & concatenate
don't deserialize

-->




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
