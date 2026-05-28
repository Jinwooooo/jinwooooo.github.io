---
title: "Tree Rotation: How AVL & Red-Black Trees Self-Balance"
date: 2024-01-12 00:00:00 +0900
categories: [Data Structure]
tags: [binary-search-tree, tree-rotation]
description: CMU Red-Black Tree Lab Insights
image:
  path: thumbnail.png
media_subpath: /assets/img/posts/2024-01-12-tree-rotation-avl-red-black-self-balance/
---

## Summary

For a balanced binary search tree (e.g. AVL, Red-Black) to stay balanced, it has to enforce certain constraints that keep the tree height near `log(n)`. The specific constraints differ between AVL and Red-Black, but both rely on **tree rotation** as the underlying primitive for restoring balance.

## Base Structure

```python
class Node():
    def __init__(self, data):
        self.data = data
        self.parent = None
        self.left = None
        self.right = None
```
{: file="Node class" }

## Rotation

![Tree rotation diagram — left/right rotation preserves in-order traversal](bst_rotate_base.png)
_Tree rotation preserves the in-order traversal_

This invariant — that rotation preserves the in-order traversal — is what makes rotation safe to use as a building block: any tree that comes out of a rotation is still a valid BST, so balancing logic only has to worry about *height*, never about correctness.

```python
# TNULL is the Red-Black sentinel (always BLACK). For just the
# rotation primitive shown here, you could replace it with None.

def rotate_left(self, curr):
    r = curr.right
    curr.right = r.left

    if r.left != self.TNULL:
        r.left.parent = curr

    r.parent = curr.parent

    if curr.parent == None:
        self.root = r
    elif curr == curr.parent.left:
        curr.parent.left = r
    else:
        curr.parent.right = r

    r.left = curr
    curr.parent = r

def rotate_right(self, curr):
    l = curr.left
    curr.left = l.right

    if l.right != self.TNULL:
        l.right.parent = curr

    l.parent = curr.parent

    if curr.parent == None:
        self.root = l
    elif curr == curr.parent.right:
        curr.parent.right = l
    else:
        curr.parent.left = l

    l.right = curr
    curr.parent = l
```
{: file="rotate_left / rotate_right" }

## Cases for Rotation

> All cases below assume the **right subtree is heavier (deeper)**, so each one rebalances by shifting weight to the left. The mirror cases (left-heavy) work symmetrically by swapping `left`/`right` in the logic.
>
> In standard AVL terminology: Cases 1 and 2 are both "right-right" (**RR**) patterns — handled by a single left rotation — distinguished here by whether the rotation changes the subtree's root height. Case 3 is the "right-left" (**RL**) pattern, which needs a double rotation.
{: .prompt-info }

### Case 1 — `x` height diff 2, `y` height diff 1

![Case 1: x has subtree height difference of 2 while y has difference of 1](case_1.png)
_Case 1 — `x` is unbalanced (Δheight = 2); a single rotation rebalances `x`, but the subtree's root height changed_

From `x`'s point of view, the two subtrees below have a height difference of 2, which means a balancing procedure must run.

After the rebalance, the subtree root's height has changed — so the procedure must recurse upward to check whether the rotation broke the parent subtree's balance.

### Case 2 — `x` height diff 2, `y` balanced

![Case 2: x has subtree height difference of 2 while y is balanced](case_2.png)
_Case 2 — `x` is unbalanced (Δheight = 2); a single rotation rebalances and the subtree's root height is unchanged_

Same trigger as Case 1, but `y` is balanced (Δheight = 0). A single rotation rebalances the subtree, **and** the root height doesn't change — so no recursion upward is needed; the parent subtree is unaffected.

### Case 3 — single rotation isn't enough; needs a double

![Case 3: x has subtree height difference of 2 requiring double rotation](case_3.png)
_Case 3 — single rotation doesn't fix the imbalance; needs a double rotation_

Δheight is still 2 at `x`, but this time the shape is such that a single rotation would just move the imbalance to the other side. A **double rotation** is required: rotate the child first, then the parent.

As in Case 1, the subtree root's height changes after rebalancing, so the procedure must recurse upward to check whether the rotation broke the parent subtree's balance.
