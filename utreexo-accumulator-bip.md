# Abstract
This document defines the process for representing, updating and verifying proofs for the Utreexo accumulator.

# License

This BIP is licensed under the BSD 3-clause license.

# Motivation

The Bitcoin UTXO set grows at a O(N) where N is each UTXO as the UTXO set is
represented with a key-value database. This is problematic as every full node
needs to keep track of the UTXO set. The Utreexo accumulator caps the growth of
the UTXO set to O(logN) as Utreexo is a collection of merkle trees.

# Design

An accumulator is a data structure that allows for the compact representation of a set. In the
Utreexo accumulator

# Specification

- `[]leaf` refers to a vector of 32 byte arrays that are to be added to the accumulator.
- `acc` refers to the Utreexo accumulator state.
- `parent_hash` is the concatenation and hash of two child hashes. If either of the child hashes are nil, the returned result is just the non-nil child by itself.

The hash function SHA512/256[^1] is used for the hash operations in the accumulator.

The Utreexo accumulator state is comprised of 2 fields:
- The roots of the merkle trees. Represented as 32 byte arrays.
- The number of total leaves added to the accumulator. Represented as uint64 field.

An Utreexo accumulator implementation MUST support these 3 operations: Add, Delete, and Verify.

## Addition

Addition adds leaves to the accumulator. The added leaves are able to be verified of their
existance with an inclusion proof.

The Addition algorithm Add(`acc`, `[]leaf`) is defined as:

For each leaf in the vector:

- From row 0 to treerows(acc.numleaves)
  - Break if there's no root at this row.
  - `parent_hash` existing root at row with `leaf`.
  - Make the result from `parent_hash` the new `leaf`.
- Increment numleaves by 1.
- Append `leaf` to `acc.roots`.

The algorithm implemented in python:

```
def add(self, leaves: [bytes]):
    for leaf in leaves:
        for row in range(tree_rows(self.numleaves)+1):
            if (self.numleaves >> row) & 1 == 0:
                break
            root = self.roots.pop()
            leaf = parent_hash(root, leaf)

        self.roots.append(leaf)
        self.numleaves += 1
```

## CalculateRoots

Both the Verification and Deletion operations depend on the Calculate Roots function.

The calculate roots algorithm Calculate_Roots(`numLeaves`, `[]hashes`, `Proof`) -> `modified_roots` is defined as:

- map Proof.targets to their hashes
- Sort Proof.targets
- Loop until Proof.targets are empty:
  - Pop off first target in Proof.targets.
  - If the the target is a root, we append to the `modified_roots` vector and continue.
  - Grab sibling hash to hash with. It's either in proof.Targets or proof.Proof.
  - Figure out if the sibling hash is on the left or the right.
  - Apply `parent_hash` to the target hash and the sibling hash.
  - Calculate parent position.
  - parent position is inserted into the sorted proof.Targets.
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

The Verification algorithm Verify(`acc`, `[]hashes`, `Proof`) -> ([]int, bool) is defined as:

- Get `modified_roots` from `Calculate_Roots(acc.numleaves, []hashes, Proof)`
- Attempt to match roots in `modified_roots` with roots in `acc`. Raise error if we don't find all the roots in the `modified_roots` in `acc`.
- Return indexes.

## Deletion

Deletion removes leaves from the accumulator. The leaves no longer exist in the accumulator.

The Deletion algorithm Delete(`acc`, `[]positions`, `Proof`) -> `acc` is defined as:

- Get `root_idx` from `Verify(acc, []positions, Proof)`. Raise error if `Verify` fails.
- Get `modified_roots` from `Calculate_Roots(acc.numleaves, []positions, Proof)`.
- Replace the matching indexes in the `modified_roots` in `acc`.

[^1] https://eprint.iacr.org/2010/548.pdf
