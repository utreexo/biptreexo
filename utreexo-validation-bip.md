# Abstract
This document defines the process for representing, updating, and validating
blocks and transactions using the Utreexo accumulator defined in bip ??? to
represent the UTXO set.

# Table of Contents
1. [License](#license)
2. [Motivation](#motivation)
3. [Design](#design)
4. [Specification](#specification)
5. [BIP0030](#bip0030)

# License
This BIP is licensed under the BSD 3-clause license.

# Motivation
When representing the UTXO set with Utreexo accumulators, additional rules are
introduced that affect the end result of the UTXO set state. In order for
Utreexo nodes to stay on the same chain as one another, it’s important the
implementations follow these new rules on top of the existing Bitcoin consensus.
This document outlines these new rules for block and tx validation as well as
updating the UTXO set state on new blocks.

# Design

There are 5 consensus critical parts of using the utreexo accumulator to
represent the UTXO set:

1: The serialized data of each UTXO ("leaf data").
2: The hash function used to hash the leaf data.
3: Which transaction outputs are excluded from the accumulator.
4: The order of operations for the additions and deletions in the accumulator.
5: The format of the UTXO proof.

A discrepancy in any of the above 5 categories will result in a different
accumulator state, leading to consensus incompatibilities.

## Specification
The serialization for all the data is in little-endian format.

#### UTXO Hash Preimages
Individual UTXOs are represented as 32 byte hashes in the Utreexo accumulator.
Below outlines the data that was used to calculate those hashes:

| Name              | Type                     | Description                               |
| ----------------- | ------------------------ | ----------------------------------------- |
| Utreexo_Tag_V1    | 64 byte array            | The version tag to be prepended to the leafhash. |
| Utreexo_Tag_V1    | 64 byte array            | The version tag to be prepended to the leafhash. |
| BlockHash         | 32 byte array            | The hash of the block in which this tx was confirmed. |
| TXID              | 32 byte array            | The transaction's TXID                    |
| Vout              | 4 bytes unsigned integer | The output index of this UTXO             |
| Header code       | 4 bytes unsigned integer | The block height and iscoinbase. This is a value obtained by left shifting the block height that confirmed this transaction, and then OR-ing it with 1, only if this transaction is a coinbase. |
| Amount            | 8 bytes unsigned integer | The amount in satoshis for this UTXO      |
| scriptPubkey size | varint                   | scriptPubKey length in bytes              |
| scriptPubkey      | variable byte array      | The locking script of the UTXO            |

##### Version tag
We use tagged hashes for the hashes committed in the accumulator for versioning
purposes. This is added so that if there are changes in the preimage of the
hash, the version tag helps to avoid misinterpretation.

The Utreexo version tag is the sha512 hash of the string "UtreexoV1". The string
"UtreexoV1" as a byte slice is "[85 116 114 101 101 120 111 86 49]". As a hex
string, it's "5574726565786f5631".  (The resulting 64 byte output is 
5b832db8ca26c25be1c542d6cceddda8c145615cff5c35727fb3462610807e20ae534dc3f64299199931772e03787d18156eb3151e0ed1b3098bdc8445861885)
(right?)

##### Blockhash
We commit to the blockhash of the block which confirms the UTXO.  This
is not currently used in the validation code, but could be used at a future
version to increase the work required for collision attacks.
A valid blockhash requires a large amount of work, which would prevent an
attacker from performing a standard cycle-finding collision attack in 2^(n/2)
operations for an n-bit hash. 

This could allow a later or alternate version to use shorter truncated hashes,
saving bandwidth and storage while still keeping Bitcoin's 2^128 security.

##### TXID
This field is added along with the vout to commit to the outpoint of a UTXO.

##### VOUT
This field is added along with the TXID to commit to the outpoint of a UTXO.

##### Header code
This field stores the block height and a boolean for marking that the utxo is
part of a coinbase transaction. Mostly serves to save space as the coinbase
boolean can be stored in a single bit.

This field is a value obtained by left shifting the block height that
confirmed this transaction, and then setting the least significant bit to 1 only
if it's part of a coinbase transaction. The code to do that is like so:

```
header_code = block_height
header_code <<= 1
if IsCoinBase {
    header_code |= 1 // only set the bit 0 if it's a coinbase.
}
```

The block height is needed as during transaction validation, it is used during
the check of BIP0065 CLTV. In current nodes, the block height is stored locally
as a part of the UTXO set. Since utreexo nodes get this data from peers, we need
to commit to the block height to avoid security vulnerabilities.

The boolean for coinbase is needed as coinbase transactions that do not have 100
confirmations is not valid and should be rejected. This data is also stored
locally as a part of the UTXO set for current nodes.

##### Amount
This field is added to commit to the value of the UTXO. With current nodes, this
is stored in the UTXO set but since we receive this in the proof from our peers,
we need to commit to this value to avoid malicious peers that may send over the
wrong amount.

##### script pubkey size
As the script pubkey is a variable length byte array, we prepend it with the
length.

##### script pubkey
This field is added to commit to the locking script of the UTXO. With current
nodes, this is stored in the UTXO set but since we receive this in the proof
from our peers, we need to commit to this value to avoid malicious peers that
may send over the wrong locking script.

#### Hash function
The leaf data is hashed with SHA-512/256, which gives us a 32 byte hash.
It was chosen over SHA-256 due to the faster performance on 64 bit systems.

#### Excluded UTXOs from the accumulator
Not all transaction outputs are added to a node's UTXO set.  Normal Bitcoin nodes
only form consensus on the set of transactions, not on the UTXO set, so different 
nodes can omit different otuputs and stay compatible as long as those outputs are
never spent.  Utreexo nodes, however, do require explicit consensus on the UTXO set
as all proofs are with respect to the merkle roots of the entire set.

For this reason, we define which UTXOs are not inserted to the accumulator.  Any 
variations here will result in Utreexo nodes with incompatible proofs.

##### Provably unspendable transaction outputs
There are outputs in the Bitcoin network that we can guarantee that they cannot
be spent without a hard-fork of the network. The following output types are not
added to the accumulator:
- Outputs that start with an OP_RETURN (0x6a)
- Outputs with a scriptPubkey larger than 10,000 bytes

##### Same block spends
Often, UTXOs are created and spent in the same block. This is allowed by Bitcoin
consensus rules as long as the output being spent is created by a transaction earlier
in the block than the spending transaction. 
In Utreexo, nodes inspect blocks and identify which outputs are being created
and destroyed in the same block, and exclude them from the accumulator and proofs.

There's no need to provide proofs for outputs which have been created in the same
block.  Adding and then immediately removing the output from the accumulator would be
possible but doesn't serve any purpose - once outputs are spent their past existence
cannot be proven with the Utreexo accumulator (and SPV proofs already provide that).

For these reasons, outputs which are spent in the same block where they are created 
are omitted from the accumulator, and those inputs are omitted from block proofs.

#### Order of operations
The Utreexo accumulator lacks associative properties during addition and the
ordering of which UTXO hash gets added first is consensus critical. For
the modification of the accumulator the steps are as follows:

1: Batch remove the UTXOs that were spent in the block based on the algorithm
   defined in bip ???.  Deletions are order-independent.
2: Batch add all non-excluded outputs in the order they're included in the
   Bitcoin block.

The removal and the addition of the hashes follow the algorithms defined in
bip ???.

#### Format of the utxo proof
The utxo proof has 2 elements: the accumulator proof and the leaf data. The 
leaf data provides the necessary utxo data for block validation that would be
stored locally for non-utreexo nodes. The accumulator proof proves that the
given utxo hash preimages are commited in the accumulator.

Accumulator proof is defined in the bip ??? and it has two elements.
1: a vector of positions of the utxo hashes in the accumulator.
2: a vector of hashes required to hash up to the roots.

For (1), the positions are in the order of the leaves that are being proved in
the accumulator. These are all the inputs in the natural blockchain order that
excludes the same block spends.

The UTXO hash preimages follow the same ordering as (1) in the accumulator
proofs. Each of the positions in (1) refer to the utxo hash preimage in the same
index.

| Field Name          | Data Type           | Byte Size | Description                             |
| ------------------- | ------------------- | --------- | --------------------------------------- |
| Accumulator Proof   | variable byte array | variable  | The utreexo proof as defined in bip ??? |
| UTXO hash preimages | variable byte array | variable  | The utxo data needed to validate all the transaction in the block |

#### Utxo proof validation
For each block, the utxo proof must be provided with the bitcoin block for
validation to be possible. Without the utxo proof, it's not possible to
validate that the inputs being referenced exists in the UTXO set.

The end result of the utxo proof validation results us in the vector of utxo
hash preimages that are required to perform the rest of the consensus
validation checks. Note that the resulting data from the utxo proof validation
is the same data that would normally be fetched from the locally stored UTXO
set.

The order of operations for the utxo proof validation are:

1: Hash the utxo preimages.
2: Verify that the utxo preimages exist in the accumulator with the verification
   algorithm specified in bip ???.

### BIP0030
BIP0030 is an added consensus check that prevents duplicate TXIDs. This check
and the historical violations of this check affect the consensus validation for
Utreexo nodes.

### BIP0030 and BIP0034 consensus check
Before BIP0030, the Bitcoin consensus rules allowed for duplicate TXIDs. If two
transactions shared a same TXID, the transaction outputs of the preceeding
transaction would overwrite the previously created UTXOs. It was assumed that
TXIDs were unique but it's trivially easy to create a transaction that share
the same TXID for coinbase transactions by re-using the same bitcoin address.

BIP0030 check is a consensus check that enforces that newly created transactions
do not have outputs that overwrites an existing UTXO.

BIP0034 was a rule where the block height was included in the script signature
of the coinbase transaction. One of the reason for the change was to make
coinbase transactions unique so that the expensive check of going through the
UTXO set wouldn't be needed. However, there were blocks in the past that had
random bytes that could be interpreted as block heights. The lowest block
heights are: 209,921, 490,897, and 1,983,702.

Up until block 209,921 the BIP0030 checks are performed for non-utreexo nodes.
Since Utreexo nodes only keep the UTXO set commitment, it's not possible to
perform the BIP0030 check.

However, it's safe to skip this check for Utreexo nodes as well since the last
checkpointed block is at height 295,000 with the block hash
00000000000000004d9b4ef50f0f9d686fd69db2e03af35a100370c64632a983. Any chain that 
doesn't include this block at height 295,000 isn't valid as removing this check
would be a hard-fork.

For block 490,897, since its coinbase was already spent in block 185,956 and
block 185,956's coinbase can only be collided with in block past 4.2 billion,
it's ok to consider this as safe as block 185,956 is also under the checkpointed
height at 295,000.

Block 1,983,702 is the first block that utreexo nodes would be in danger of a
consensus failure due to the inability to perform the BIP0030 checks. However,
this block will happen in roughly 21 years from now and mitigation of the
BIP0030 checks are defined in bip ??? and suggested as a soft-fork.

### Historical BIP0030 violations
There were two UTXOs that were overwritten due to this consensus rule are:
e3bf3d07d4b0375638d5f1db5255fe07ba2c4cb067cd81b84ee974b6585fb468:0 at block height 91,722
d5d27987d2a3dfc724e359870c6644b40e497bdc0589a033220fe15429d88599:0 at block height 91,812

Since the leaf hashes that are committed to the utreexo accumulator commit to
the block hash as well, all the leaf hashes are unique and the two historical
violations do not happen with how the UTXO set is represented with the utreexo
accumulator. To be consensus compatible with clients that do have the historical
violations, the leaves representing these two UTXOs in the utreexo accumulator
are hardcoded as unspendable.

These two leaf hashes encoded in hex string are:

1: 84b3af0783b410b4564c5d1f361868559f7cf77cfc65ce2be951210357022fe3
2: bc6b4bf7cebbd33a18d6b0fe1f8ecc7aa5403083c39ee343b985d51fd0295ad8

(1) represents the UTXO created at block height 91,722 and (2) represents the
UTXO created at block height 91,812.
