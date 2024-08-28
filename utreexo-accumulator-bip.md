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

## Positions in the accumulator

Each of the hashes in the Utreexo accumulator can be referred by a position represented by uint64.
The positions are based off of the total amount of rows the accumulator was allocated for. The
positions are numbered from 0 to 2^N where N is the number of total rows in the accumulator.
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
the accumulator resizing to hold 2^(N+1) leaves. For example, when adding a leaf to the accumulator
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

The new accumulator with all the positions:

```
30
|-------------------------------\
28                              29
|---------------\               |---------------\
24              25              26              27
|-------\       |-------\       |-------\       |-------\
16      17      18      19      20      21      22      23
|---\   |---\   |---\   |---\   |---\   |---\   |---\   |---\
00  01  02  03  04  05  06  07  08  09  10  11  12  13  14  15
```

# Definitions

- `hash` refers to a vector of 32 byte arrays.
- `[]hash` refers to a vector of `hash`.
- `acc` refers to the Utreexo accumulator state.

`acc` is comprised of 2 fields:
  - `roots` refers to the roots of the merkle trees. Represented as `[]hash`.
  - `numleaves` refers to the number of total leaves added to the accumulator. Represented as uint64.

`proof` is an inclusion proof for elements in the accumulator. It's comprised of two fields:
  - `targets` are the positions of the elements being proven. Represented as a vector of uint64.
  - `proof` are the hashes needed to hash the roots. Represented as a `[]hash`.
- `proof.proof` MUST be in ascending order. The proof is considered invalid otherwise.

# Specification

The hash function SHA512/256[^2] is used for the hash operations in the accumulator.

The following utility functions are required for the accumulator operations:

*parent_hash(left, right)* is the concatenation and hash of two child hashes. If either of the child hashes are nil, the returned result is just the non-nil child by itself.

*treerows(numleaves)* is a function that returns the minimum number of bits needed to represent of `numleaves`-1. Returns 0 if `numleaves` is 0.

*parent(position, total_rows)* is a function that returns the parent position of the given position in an accumulator with leaf count of `numleaves`.
```
(position >> 1) | (1 << total_rows)
```

*root_position(numleaves, row, total_rows)* is a function that returns the position of the root at the given row. Will return a garbage value if there's no root at the row.
```
mask = (2 << total_rows) - 1
before = numleaves & (mask << (row + 1))
shifted = (before >> row) | (mask << (total_rows + 1 - row))
shifted & mask
```

*root_present(numleaves, row)* returns if there's a root at the given row in an accumulator with leaf count of `numleaves`.
```
numleaves & (1 << row) != 0
```

*detect_row(position, total_rows)* is a function that returns which row the given position is at in an accumulator with row count of total_rows.
```
for row in range(total_rows, -1, -1):
    rowbit = 1 << row
    if rowbit & position == 0: return total_rows-row
```

*isroot(position, numleaves, total_rows)* returns if the position is a root in an accumulator with leaf count of `numleaves` and a row count of total_rows.
```
row = detect_row(position, total_rows)
root_present(numleaves, row) && position == root_position(numleaves, row, total_rows)
```

*is_right_sibling(position)* returns true if the position is on the right side.
```
position & 1 == 1
```

*right_sibling(position)* returns the position of the sibling on the right side. Returns itself if the position is on the right.
```
position | 1
```

*getrootidxs(numleaves, positions)* returns the indexes of the roots in the accumulator state that will be modified when deleting the given positions. Returned indexes are in descending order.

An Utreexo accumulator implementation MUST support these 3 operations: Add, Verify, and Delete.

## Addition

Addition adds a leaf to the accumulator. The added leaves are able to be verified of their
existance with an inclusion proof.

- Inputs:
  - The utreexo accumulator state.
  - 32 byte array to be added.

The Addition algorithm Add(`acc`, `hash`) is defined as:

- From row 0 to and including `treerows(acc.numleaves)`
  - Break if there's no root at this row.
  - *parent_hash* existing root at row with `hash`.
  - Make the result from *parent_hash* the new `hash`.
- Increment acc.numleaves by 1.
- Append `hash` to acc.roots.

The algorithm implemented in python:

```
def add(self, hash: bytes):
    for row in range(tree_rows(self.numleaves)+1):
        if not root_present(self.numleaves, row): break
        root = self.roots.pop()
        hash = parent_hash(root, hash)

    self.roots.append(hash)
    self.numleaves += 1
```

## CalculateRoots

Both the Verification and Deletion operations depend on the Calculate Roots function.

- Inputs:
  - The total number of leaves added to the accumulator state.
  - `[]hash` that are the hashes for the `proof.targets`.
  - `proof`.

The passed in `[]hash` and `proof.targets` should be in the same order. The element at index i in []hashes should
be the hash for element at index i in `proof.targets`. Otherwise the returned calculated_roots will be invalid.

The calculate roots algorithm is defined as CalculateRoots(numleaves, `[]hash`, `proof`) -> calculated_roots:

- Check if length of `proof.targets` is equal to the length of `[]hash`. Return early if they're not equal.
- map proof.targets to their hash.
- Sort proof.targets.
- Loop until proof.targets are empty:
  - Pop off the first target in proof.targets. Pop off the associated hash as well.
  - If the the target is a root, we append to the calculated_roots vector and continue.
  - Grab sibling hash to hash with. It'e either in proof.targets or proof.proof.
  - Figure out if the sibling hash is on the left or the right.
  - Apply *parent_hash* to the target hash and the sibling hash.
  - Calculate parent position.
  - parent position is inserted into the sorted proof.targets.
  - parent hash is mapped to the parent position.

- Return calculated_roots

The algorithm implemented in python:

```
def calculate_roots(numleaves: int, dels: [bytes], proof: Proof) -> [bytes]:
    if not proof.targets: return []
    if len(proof.targets) != len(dels): return []

    position_hashes = {}
    for i, target in enumerate(proof.targets):
        position_hashes[target] = None if dels is None else dels[i]

    calculated_roots = []
    sortedTargets = sorted(proof.targets)
    while sortedTargets:
        pos = sortedTargets.pop(0)
        cur_hash = position_hashes.pop(pos)

        if isroot(pos, numleaves, tree_rows(numleaves)):
            calculated_roots.append(cur_hash)
            continue

        parent_pos, p_hash = parent(pos, tree_rows(numleaves)), bytes
        if sortedTargets and right_sibling(pos) == sortedTargets[0]:
            sib_pos = sortedTargets.pop(0)
            p_hash = parent_hash(cur_hash, position_hashes.pop(sib_pos))
        else:
            proofhash = proof.proof.pop(0)
            p_hash = parent_hash(proofhash, cur_hash) if is_right_sibling(pos) else parent_hash(cur_hash, proofhash)

        position_hashes[parent_pos] = p_hash
        bisect.insort(sortedTargets, parent_pos)

    return calculated_roots
```

## Verification

- Inputs:
  - The accumulator state.
  - `[]hash` that are the hashes for the `proof.targets`.
  - `proof`.

The Verification algorithm Verify(`acc`, `[]hash`, `proof`) is defined as:

- Raise error if length of `[]hash` differ from `proof.targets`.
- Get modified_roots from CalculateRoots(acc.numleaves, []hash, Proof).
- Get root_idxs from *getrootidxs*.
- Raise error if the length of modified_roots and root_idxs do not match.
- Attempt to match roots in modified_roots with roots in `acc`. Raise error if we don't find all the roots in the modified_roots in `acc`.

The algorithm implemented in python:

```
def verify(self, dels: [bytes], proof: Proof):
    if len(dels) != len(proof.targets):
        raise("len of dels and proof.targets differ")

    root_candidates = calculate_roots(self.numleaves, dels, proof)
    root_idxs = getrootidxs(self.numleaves, proof.targets)

    if len(root_candidates) != len(root_idxs):
        raise("length of calculated roots from the proof and expected root count differ")

    for i, idx in enumerate(root_idxs):
        if self.roots[idx] != root_candidates[i]:
            raise("calculated roots from the proof and matched roots differ")
```

## Deletion

Deletion removes leaves from the accumulator. The deletion algorithm takes in a `proof` but it does not
verify that the proof is valid. It assumes that the passed in proof has already passed verification.

- Inputs:
  - The accumulator state.
  - `proof`.

The Deletion algorithm Delete(`acc`, `Proof`) -> `acc` is defined as:

- Get the modified indexes of the roots `root_idxes` from `getrootidxs`.
- Get modified_roots from Calculate_Roots(acc.numleaves, []positions, Proof).
- Replace the matching indexes from the root_idxes in `acc.roots` with modified_roots.

The algorithm implemented in python:

```
def delete(self, proof: Proof):
    modified_roots = calculate_roots(self.numleaves, None, proof)
    root_idxs = getrootidxs(self.numleaves, proof.targets)
    for i, idx in enumerate(root_idxs):
        self.roots[idx] = modified_roots[i]
```

# Reference Implementations

In Python - https://github.com/utreexo/pytreexo

In Rust - https://github.com/mit-dci/rustreexo

In Go - https://github.com/utreexo/utreexo

[^1]: https://eprint.iacr.org/2015/718
[^2]: https://eprint.iacr.org/2010/548.pdf

