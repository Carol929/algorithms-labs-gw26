---
layout: default
title: Lab 5
nav_order: 6
---

# CSCI 3212 Lab 5: AVL Tree Deletion and Rebalancing

In this lab, you will extend your AVL tree implementation from Lab 4 by implementing
deletion with post-deletion rebalancing. You will trace and implement AVL deletion,
analyze why deletions require more complex rebalancing than insertions, and
empirically compare insertion and deletion cost.

This lab reuses the pointer-based linked AVL trees and rotation infrastructure
from Lab 4. Deletion follows the same three BST deletion cases (0, 1, 2 children),
then rebalances every ancestor of the deleted node on the way up toward the root.
Unlike insertion, a single deletion can trigger **multiple independent rotations**
at different ancestors.

## Files and deliverables

| File | Your work |
|---|---|
| `README.md` | Complete the trace tables and written responses in your lab notes or a copy of this file |
| `avl_practice.py` | Implement `avl_delete` and complete the rebalancing loop; rotation functions from Lab 4 are provided |
| `lab_checks.py` | Provided checks and profiling demonstration; do not edit |

- [ ] Part 1: AVL deletion strategy, rebalancing pass conceptual understanding
- [ ] Part 2: Deletion traces (single rotation, double rotation, multiple rotations)
- [ ] Part 3: Implement `avl_delete` with post-deletion rebalancing
- [ ] Part 4: Analyze and compare insertion vs. deletion cost
- [ ] Run the practice file and resolve all failed checks.

Keep the function names and parameters unchanged. The provided checks inspect
pointer identities, in-order traversals, parent references, node heights, and
balance factors directly.

---

## Part 1: AVL Deletion Strategy

### Why deletion is harder than insertion

In Lab 4, AVL insertion was structured as: **insert → walk ancestors up → fix at most one violation**.
A single insertion creates a single "problem zone" (the inserted key's ancestors),
and one rotation fixes the entire subtree.

AVL deletion is fundamentally different:

1. **Multiple violation zones:** Deleting a node can cause imbalances at multiple
   ancestors simultaneously.
2. **Cascading rebalancing:** After fixing an imbalance at ancestor $z$ with a rotation,
   the rotated subtree may have a different height than before. This can cause a new
   imbalance higher up.
3. **Propagate further:** Unlike insertion (which stops after one rotation), deletion
   must check every ancestor all the way to the root. After each rotation, the
   rebalancing loop continues.

**Key insight:** An insertion at height $h$ changes the subtree's height by at most 1
locally, stopping rebalancing immediately. A deletion can propagate height changes
all the way to the root.

### Post-deletion rebalancing strategy

```text
AVL-DELETE(T, key)
  z = BST-DELETE(T, key)          // Perform BST deletion; z is the deleted node (or None)
  current = parent_of_deleted     // Start rebalancing from the parent of the deleted node
  while current != None
    UPDATE-HEIGHT(current)        // Recompute height after structural change
    bf = BALANCE-FACTOR(current)
    if |bf| >= 2                  // Imbalance detected
      // Determine which case (LL, RR, LR, RL) and rotate
      // Unlike insertion, the key is NOT available—use bf signs instead
      if bf > 1                   // Left-heavy
        if BALANCE-FACTOR(current.left) >= 0
          ROTATE-RIGHT(T, current)          // LL
          current = current.parent          // Move up after rotation
        else
          ROTATE-LEFT-RIGHT(T, current)     // LR
          current = current.parent          // Move up after rotation
      else if bf < -1             // Right-heavy
        if BALANCE-FACTOR(current.right) <= 0
          ROTATE-LEFT(T, current)           // RR
          current = current.parent          // Move up after rotation
        else
          ROTATE-RIGHT-LEFT(T, current)     // RL
          current = current.parent          // Move up after rotation
    current = current.parent      // Continue to next ancestor
  return z
```

**Critical difference from insertion:** After a rotation in insertion, the rebalancing
stops immediately. In deletion, we must continue up the tree. The rotated subtree may
have a different height, creating imbalances higher up.

### 1.1 Short answer: BST deletion reminder

**TODO 1.1:** Briefly recall the three deletion cases from Lab 3/4:
- What happens when the target node has 0 children?
- What happens when the target node has 1 child?
- What happens when the target node has 2 children, and why is the in-order successor used?

- 0 children: just remove the node. Its parent's pointer becomes `None`.
- 1 child: the child takes the node's place (transplant).
- 2 children: replace the node with its in-order successor, the minimum of its right subtree. It is the next larger key, so BST order stays correct, and it has no left child, so removing it from its old spot is an easy 0 or 1 child case.

### 1.2 Short answer: Height change after deletion

**TODO 1.2:** When you delete a leaf node from an AVL tree:
- Does the leaf's parent's height change? By how much?
- Can the grandparent's height change?
- Can the imbalance propagate to the root?

- The parent's height drops by 1 if the leaf was its only child. Otherwise it stays the same.
- Yes. If the parent's height drops, the grandparent's height can drop too.
- Yes. Each fix can make a subtree 1 shorter, so the change can keep moving up and cause imbalances all the way to the root.

---

## Part 2: AVL Deletion Traces

### Example: AVL trees for deletion traces

For the traces below, we use AVL trees built carefully so that deletions trigger imbalances.

### 2.1 Trace: Single rotation after deletion

Start with this AVL tree:
```
      30
     /  \
   20    40
   /
  10
```
(All nodes balanced: 30 has BF=1, 20 has BF=1, others BF=0.)

**TODO 2.1:** Delete key `40` from this tree. Trace the rebalancing:

1. Perform BST deletion of 40 (it's a leaf). What is the tree after deletion?
2. Rebalance from the parent of the deleted node (30).
3. What is the balance factor at 30?
4. Identify the violation signature (LL, RR, LR, or RL) and the required rotation.
5. After rotation, is the tree still imbalanced? If so, continue rebalancing.
6. Draw the final tree and record the in-order traversal.

| Step | Action | Tree state | Unbalanced node | BF | Signature | Rotation | Notes |
|---|---|---|---|---|---|---|---|
| 1 | Delete 40 | 40 is removed (leaf) | - | - | - | - | Tree now has 30 root, 20 left, nothing right |
| 2 | Rebalance from 30 | `30` -> left `20` -> left `10` | `30` | 2 | LL | Right rotation at `30` | `20` has BF = 1 (left-heavy), so one single rotation fixes it |
| 3 | After rotation | `20` root, `10` left, `30` right | none | all 0 | - | - | Final state |

In-order: [10, 20, 30]

### 2.2 Trace: Double rotation after deletion

Start with this AVL tree:
```
      30
     /  \
   10    40
    \
    20
```
(All nodes balanced: 30 has BF=0, 10 has BF=-1, others BF=0.)

**TODO 2.2:** Delete key `40` from this tree. Trace the rebalancing:

1. Perform BST deletion of 40 (it's a leaf).
2. Rebalance from the parent of the deleted node (30).
3. What is the balance factor at 30 after 40 is deleted?
4. Identify the violation signature. Is node 10 left-heavy or right-heavy?
5. Which rotation(s) are needed (single or double)?
6. Draw the final tree and record the in-order traversal.

| Step | Action | Current node | BF before | Signature | Rotation applied | BF after |
|---|---|---|---|---|---|---|
| 1 | Delete 40 | 30 | 2 | LR | Double: left rotation at `10`, then right rotation at `30` | 0 |
| 2 | Verify final | - | - | - | - | - |

`10` has BF = -1, so it is right-heavy. Final tree: `20` root, `10` left, `30` right. In-order: `[10, 20, 30]` 

### 2.3 Trace: Two-child deletion with rebalancing

Start with this AVL tree:
```
        50
       /  \
      30   70
     / \     \
   20  40    80
   /
  10
```
(All balanced initially.)

**TODO 2.3:** Delete key `30`. This is a 2-child deletion (has both 20 and 40 as children).
Trace the rebalancing:

1. Find the in-order successor of 30 (minimum of right subtree: 40).
2. Perform the transplant: replace 30 with 40, move 40's children appropriately.
3. Rebalance from the appropriate starting node (the parent of where 40 was removed).
4. At each step, identify any violation and apply the necessary rotation.
5. Continue until no more imbalances exist.

| Step | Current node | BF | Imbalanced? | Violation | Rotation applied |
|---|---|---|---|---|---|
| 1 | `40` (after replacing 30 with 40) | 2 | Yes | LL (`20` has BF = 1) | Right rotation at `40` |
| 2 | `50` (continue up) | 0 | No | none | none |

`40` was 30's direct right child, so rebalancing starts at `40` itself. Final tree: `50` root, left `20` (children `10`, `40`), right `70` (right child `80`). In-order: `[10, 20, 40, 50, 70, 80]`

---

## Part 3: Implementation

Open `avl_practice.py` and implement the deletion function:

### 3.1 Implement AVL deletion

**TODO 3.1:** Complete `avl_delete(tree, key)` in `avl_practice.py`.

The skeleton is provided. Complete the rebalancing loop to:
1. Identify the parent of the deleted node to start rebalancing from.
2. Walk up from that node to the root, checking and fixing each ancestor.
3. Return the deleted node (or `None` if key not found).

Your implementation must:
- Correctly identify the rebalancing start point for all three BST deletion cases.
- Update heights and balance factors as you walk up.
- Recognize and apply the correct rotation for each violation signature (LL, RR, LR, RL).
- **Continue rebalancing at every ancestor** (unlike insertion, which stops after one rotation).

Provided helpers (already implemented):
- `transplant(tree, u, v)` - updates tree pointers
- `tree_minimum(node)` - finds minimum in subtree
- `tree_search(node, key)` - searches for key
- `balance_factor(node)` - returns BF
- `rotate_left(tree, node)`, `rotate_right(tree, node)` - single rotations
- `rotate_left_right(tree, node)`, `rotate_right_left(tree, node)` - double rotations
- `update_height(node)` - recalculates node's height

```bash
python3 avl_practice.py
```

---

## Part 4: Insertion vs. Deletion Comparison

Once deletion is working, the test suite runs a profiling experiment:
insert and delete 1000 random keys into an AVL tree,
measuring the number of rotations triggered by each operation.

### 4.1 Short answer: Why is deletion costlier?

**TODO 4.1:** Based on your implementation and understanding of the algorithm:

1. Why can a single deletion trigger multiple rotations at different ancestors,
   whereas a single insertion triggers at most one rotation?
2. What property of rotations ensures that insertion stops after one fix?
3. Does a deletion ever need to rebalance higher than the root? Explain.

1. Insertion makes a subtree taller, and one rotation brings it back to its old height, so nothing above changes. Deletion makes a subtree shorter, and a rotation can leave it 1 shorter than before, so the next ancestor can become unbalanced too. This can repeat at every level.
2. After an insertion, the rotation restores the subtree to the exact height it had before the insert, so the ancestors above see no change.
3. No. The root has no parent, so the loop stops there. At most one fix per level means at most O(log n) rotations.

### 4.2 Short answer: Real-world implications

**TODO 4.2:** Consider a scenario where an application frequently insertions and deletions
in an AVL tree (e.g., a priority queue or cache).

1. Based on the rotation cost, would you expect insertions or deletions to be slower?
2. If deletions become a bottleneck, what alternative data structure (from this course)
   might handle deletions more efficiently?

1. Deletions, because one deletion can cause several rotations while an insertion causes at most one. Both are still O(log n).
2. A hash table (Lab 6) deletes in O(1) average time when we only need lookups by key, like a cache. For a priority queue, a binary heap removes the min in O(log n) with no rotations.

---

## Final check

Run the practice file from within the `lab5/` directory:

```bash
python3 avl_practice.py
```

- Any unfinished function reports `[TODO]`.
- Any logic error or failed assertion reports `[FAIL]`.
- Any fully working function reports `[PASS]`.

The practice file exits with a nonzero exit code if any check is unfinished or
failing. When all checks pass, the command returns exit code `0`.

The profiling output compares insertion vs. deletion rotation counts on random keys
and provides empirical evidence of why deletion is costlier.
