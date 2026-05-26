---
title: Repurposing Slack Bits and Freed Blocks
date: 2024-02-22 00:00:00 +0900
categories: [Data Structure]
tags: [malloc, memory-management]
description: CMU Malloc Lab Insights
image:
  path: thumbnail.png
media_subpath: /assets/img/posts/2024-02-22-repurposing-slack-bits-and-freed-blocks/
---

## Summary

Some insights after 2 weeks of Malloc Lab (skeletal code from CMU CS:APP).

- **Header and Footer** — repurpose the bits the higher-level application can't reach.
- **Explicit Free Block Management** — embed bookkeeping structures inside freed blocks themselves to speed up allocator lookups.

## Header and Footer

```
W/O FOOTER

    4 bytes        4+ bytes
|--------------|--------------|
|    HEADER    |    PAYLOAD   |
|--------------|--------------|

W/ FOOTER

    4 bytes               8+ bytes                4 bytes
|--------------|--------------|--------------|--------------|
|    HEADER    |           PAYLOAD           |    FOOTER    |
|--------------|--------------|--------------|--------------|
```
{: file="Block layout: with vs without footer" }

The skeletal code from Malloc Lab (`mm.c`) uses 4 bytes for the header. The natural question is: why 4 bytes? After some digging — the Malloc Lab is essentially a sandbox that reserves around 20MB of heap space (i.e. a virtual environment). Since 32 bits can encode values up to 4GB, 4 bytes is more than enough to represent any block size within that sandbox.

```
00000000 00000000 00000000 00000[000]  <- 3 LSB
// when malloced, the 3 LSB cannot be used because the min block size is 8
// ! it does not have to be 3; it just makes sense in this case
```
{: file="3 LSB unused due to 8-byte alignment" }

Because the alignment is 8 bytes, every block size is a multiple of 8 — so the 3 LSB of the encoded size field are always `0` and can be repurposed for flags. The skeletal code uses one of those 3 LSB to mark whether the block is allocated (`1`) or free (`0`). That leaves 29 bits for the size — enough to represent up to 512MB. In practice, individual block sizes in the lab never come close to that ceiling (the lab caps allocations well below the 20MB total), so the top ~6 MSB also go unused. Combined with the 2 still-free LSB, the header has roughly 8 bits of metadata budget left for further optimization.

```
[000000]{00 00000000 00000000 00000}[00(0)]
// [] = bits unusable due to application constraints
// () = bit indicating whether the block is free or allocated
// {} = bits encoding the block size in bytes
```
{: file="Bit roles within the header/footer word" }

> **Sidetrack** — looking at the skeletal code, the footer is only used in coalesce case 3 and case 4. It is worth considering removing the footer entirely if the custom `mm.c` does not coalesce frequently (e.g. a lazy coalesce that sweeps from the heap head only when needed).
{: .prompt-info }

## Explicit Free Block Management

Before properly learning about explicit free block management, my first thought was to keep a data structure *outside* the custom malloc sandbox to track free list blocks (which is completely wrong …). Explicit free block management instead embeds the tracking data structure *within* the sandbox itself — by reusing the unused space inside free blocks (ingenious!).

There are multiple policies for handling free blocks explicitly. Two are covered below: Doubly Linked List and Segregated List.

### Doubly Linked List (optionally circular)

```
// in CS:APP, PREV_BLK = PREDECESSOR and NEXT_BLK = SUCCESSOR

    4 bytes        4 bytes        4 bytes           8+ bytes          4 bytes
|--------------|--------------|--------------|-------------------|--------------|
|    HEADER    |   PREV_BLK   |   NEXT_BLK   |   TRASH_PAYLOAD   |    FOOTER    |
|--------------|--------------|--------------|-------------------|--------------|
```
{: file="Free Block Layout Visualized" }

The simplest policy for explicit free block management is the doubly linked list. When a block is freed, the `PREV_BLK` and `NEXT_BLK` pointers are written inside the now-unused payload, linking it into the previous and next free blocks. This lets the allocator walk only the free blocks (skipping allocated ones) when servicing a `malloc`.

With implicit free block management, the allocator walks every block in the heap — free and allocated alike — reading each header to check both the allocation flag and the size. With an explicit doubly linked list, the allocator skips the "is it free?" check entirely: it just walks the free list and checks whether each free block is large enough for the payload being inserted.

Insertion policy (LIFO, FIFO, address-ordered) is independent of whether the list is circular. The default is **LIFO**: just prepend each newly-freed block to the head — works on any DLL. **FIFO** needs to insert at the tail, which means tracking a tail pointer or making the list circular (so the tail is `head->prev`). The right policy depends on the allocator's goals: LIFO favors cache locality on recently-freed blocks, while FIFO and address-ordered tend to fragment less.

### Segregated List

```
// each head points to the start of a free list holding blocks within a size class.
// if no block of the requested size is available, fall back to the next larger class.

    4 bytes          4 bytes         4 bytes         n bytes
|---------------|---------------|---------------|--------------|
|    <16 BYTE   |    <32 BYTE   |    <64 BYTE   |      ...     |
|---------------|---------------|---------------|--------------|
        |               |
        ▼               ▼
  |-----------|   |-----------|
  | <16 BLK_1 |   | <32 BLK_2 |        ...
  |-----------|   |-----------|
        |               .
        ▼               .
  |-----------|
  | <16 BLK_2 |
  |-----------|
        .
        .

// note: blocks within each class are linked like a doubly linked list
```
{: file="Segregated List Visualized" }

Unlike the naive doubly linked list — which traverses every free block while checking each one's size — the segregated list adds a central index whose entries point to the start of a free list holding blocks within a given size class. This adds overhead, but reduces the time required to locate a free block of the appropriate size.
