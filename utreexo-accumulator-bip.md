# Abstract
This document defines the process for representing, updating and verifying proofs for the Utreexo accumulator.

# License

This BIP is licensed under the BSD 3-clause license.

# Motivation

The Bitcoin UTXO set grows at a O(N) where N is each UTXO as the UTXO set is
represented with a key-value database. This is problematic as every full node
needs to keep track of the UTXO set. The Utreexo accumulator caps the growth of
the UTXO set to O(log2N) as Utreexo is a collection of merkle trees.

# Design

An accumulator is a data structure that allows for the compact representation of a set. The Utreexo
accumulator uses an append only merkle tree design found in [^1] and extends the work to make deletions
possible in O(log2N).

# Specification

- `[]leaf` refers to a vector of 32 byte arrays that are to be added to the accumulator.
- `acc` refers to the Utreexo accumulator state.
- `parent_hash` is the concatenation and hash of two child hashes. If either of the child hashes are nil, the returned result is just the non-nil child by itself.
- The hash function SHA512/256[^2] is used for the hash operations in the accumulator.
- The Utreexo accumulator state is comprised of 2 fields:
  - `roots` refers to the roots of the merkle trees. Represented as 32 byte arrays.
  - `numleaves` refers to the number of total leaves added to the accumulator. Represented as uint64 field.
- `treerows` is a function that returns the popcount of `numleaves` in the accumulator state. 0 if `numleaves` is 0.

An Utreexo accumulator implementation MUST support these 3 operations: Add, Verify, and Delete.

## Positions in the accumulator

Each of the hashes in the Utreexo accumulator can be referred by a position represented by uint64.
The positions are based off of the total amount of rows the accumulator was allocated for. The
positions are numbered from 0 to 2^$ where N is the number of total rows in the accumulator.
Elements in each of the row are numbered in ascending order, moving up to the leftmost element in
the row above.

For example, an accumulator state with N=3 will look like so:

```
14
|---------------\
12              13
|-------\       |-------\
08      09      10      11
|---\   |---\   |---\   |---\
00  01  02  03  04  05  06  07
```

It's possible to have less than 2^N elements in the accumulator. For an accumulator state where
N=3 but with 6 elements will look like so:

```

|---------------\
12
|-------\       |-------\
08      09      10
|---\   |---\   |---\   |---\
00  01  02  03  04  05
```

When adding another leaf to the accumulator when it's already allocated 2^N leaves will result in
the accumulator resizing to hold 2^N+1 leaves. For example, when adding a leaf to the accumulator
state here:

```
14
|---------------\
12              13
|-------\       |-------\
08      09      10      11
|---\   |---\   |---\   |---\
00  01  02  03  04  05  06  07
```

The new accumulator will look like so:

```

|-------------------------------\
28
|---------------\               |---------------\
24              25
|-------\       |-------\       |-------\       |-------\
16      17      18      19
|---\   |---\   |---\   |---\   |---\   |---\   |---\   |---\
00  01  02  03  04  05  06  07  08
```

## Addition

Addition adds leaves to the accumulator. The added leaves are able to be verified of their
existance with an inclusion proof.

The Addition algorithm Add(`acc`, `[]leaves`) is defined as:

For each leaf in the vector:

- From row 0 to `treerows(acc.numleaves)`
  - Break if there's no root at this row.
  - `parent_hash` existing root at row with `leaf`.
  - Make the result from `parent_hash` the new `leaf`.
- Increment `acc.numleaves` by 1.
- Append `leaf` to `acc.roots`.

The algorithm implemented in python:

```
class Stump:
    def __init__(self):
        self.numleaves = 0
        self.roots = []

def add(acc: Stump, leaves: [bytes]):
    for leaf in leaves:
        for row in range(tree_rows(acc.numleaves)+1):
            if (acc.numleaves >> row) & 1 == 0:
                break
            root = acc.roots.pop()
            leaf = parent_hash(root, leaf)

        acc.roots.append(leaf)
        acc.numleaves += 1
```

## Proof

- proof is an inclusion proof for elements in the accumulator. It's comprised of two fields:
  - `targets` are the positions of the elements being proven. Represented as a `[]uint64`.
  - `proof` are the hashes needed to hash the roots. Represented as a `[]uint64`.

- `proof.proof` must be in ascending order. The proof is considered invalid otherwise.

## CalculateRoots

Both the Verification and Deletion operations depend on the Calculate Roots function.

The calculate roots algorithm is defined as CalculateRoots(`numleaves`, `[]hashes`, `proof`) -> `modified_roots`.
The passed in `[]hashes` and `proof.targets` should be in the same order.
The element at index 0 in []hashes should be the hash for element at index 0 in `proof.targets`.

- Check if length of `proof.targets` is equal to the length of `[]hashes`. Return early if they're not equal.
- map proof.targets to their hashes
- Sort proof.targets
- Loop until proof.targets are empty:
  - Pop off first target in proof.targets.
  - If the the target is a root, we append to the `modified_roots` vector and continue.
  - Grab sibling hash to hash with. It'e either in proof.targets or proof.proof.
  - Figure out if the sibling hash is on the left or the right.
  - Apply `parent_hash` to the target hash and the sibling hash.
  - Calculate parent position.
  - parent position is inserted into the sorted proof.targets.
  - `parent_hash` is mapped to the parent position.

- Return modified_roots

The algorithm implemented in python:

```
def calculate_roots(numleaves: int, dels: [bytes], proof: Proof) -> [bytes]:
    if not proof.targets:
        return []

    calculated_roots = []

    posHash = {}
    for i, target in enumerate(proof.targets):
        if dels is None:
            posHash[target] = None
        else:
            posHash[target] = dels[i]

    sortedTargets = sorted(proof.targets)
    while sortedTargets:
        pos = sortedTargets.pop(0)
        cur_hash = posHash[pos]
        del posHash[pos]

        if isroot(pos, numleaves, tree_rows(numleaves)):
            calculated_roots.append(cur_hash)
            continue

        parent_pos = parent(pos, tree_rows(numleaves))
        bisect.insort(sortedTargets, parent_pos)

        if sortedTargets and pos | 1 == sortedTargets[0]:
            sib_pos = sortedTargets.pop(0)
            posHash[parent_pos] = parent_hash(cur_hash, posHash[sib_pos])

            del posHash[sib_pos]
        else:
            proofhash = proof.proof.pop(0)

            if pos & 1 == 0:
                posHash[parent_pos] = parent_hash(cur_hash, proofhash)
            else:
                posHash[parent_pos] = parent_hash(proofhash, cur_hash)

    return calculated_roots
```

## Verification

The Verification algorithm Verify(`acc`, `[]hashes`, `proof`) -> ([]int, bool) is defined as:

- Get `modified_roots` from `CalculateRoots(acc.numleaves, []hashes, Proof)`
- Attempt to match roots in `modified_roots` with roots in `acc`. Raise error if we don't find all the roots in the `modified_roots` in `acc`.
- Return indexes of the changed modified roots.

```
def verify(acc: Stump, dels: [bytes], proof: Proof) -> [int]:
    if len(dels) != len(proof.targets):
        raise("len of dels and proof.targets differ")

    root_candidates = calculate_roots(acc.numleaves, dels, proof)

    root_idxs = []
    for i in range(len(acc.roots)):
        j = len(acc.roots) - (i+1)
        if len(root_candidates) > len(root_idxs):
            if acc.roots[j] == root_candidates[len(root_idxs)]:
                root_idxs.append(j)

    if len(root_idxs) != len(root_candidates):
        raise("calculated roots from the proof and matched roots differ")

    return root_idxs
```

## Deletion

Deletion removes leaves from the accumulator. The leaves no longer exist in the accumulator.

The Deletion algorithm Delete(`acc`, `[]positions`, `Proof`) -> `acc` is defined as:

- Get `root_idx` from `Verify(acc, []positions, Proof)`. Raise error if `Verify` fails.
- Get `modified_roots` from `Calculate_Roots(acc.numleaves, []positions, Proof)`.
- Replace the matching indexes from the `root_idx` in `acc.roots` with `modified_roots`.

```
def delete(acc: Stump, dels: [bytes], proof: Proof):
    dels_copy = dels.copy()
    proof_copy = copy.copy(proof)
    root_idxs = acc.verify(dels_copy, proof_copy)

    modified_roots = calculate_roots(acc.numleaves, None, proof)

    for i, idx in enumerate(root_idxs):
        acc.roots[idx] = modified_roots[i]
```

[^1]: https://eprint.iacr.org/2015/718
[^2]: https://eprint.iacr.org/2010/548.pdf
