---
title: "Data Modeling for KIA DCE: Hierarchies, Children, and Draft/Publish"
date: 2025-11-02 00:00:00 +0900
categories: [Data Structure]
tags: [hierarchical-data, draft-publish]
description: Domain modeling notes from the KIA DCE project — a hierarchical division tree, a parent-children content model, and a draft/publish state machine
mermaid: true
image:
  path: thumbnail.png
media_subpath: /assets/img/posts/2025-11-02-kia-dce-data-architecture/
---

## Summary

- **KIA Users** → hierarchical division tree (5 levels, parent-linked)
- **Content** → parent document with 1:1 typed children
- **Editing & visibility** → draft/publish model

The KIA user hierarchy is given by the client side — it mirrors KIA's **DDMS** (Dealer Distributor Management System). The content management structure was designed through back-and-forth among client, designer, and developer.

## KIA Users → DCE Divisions

KIA's DDMS defines 5 user types:

HQ
: Headquarters — the brand itself (KIA)

RHQ
: Regional Headquarters — oversees a multi-country region

NSC
: National Sales Company — KIA's per-country subsidiary

Distributor
: Imports and wholesales within a country

Dealer
: Customer-facing retail outlet

At its most basic, the schema looks like this:

![Division hierarchy tree — HQ at the root, then RHQ, NSC, Distributor, Dealer with sample divisionIds](kia-division-tree.png)
_Division hierarchy: HQ → RHQ → NSC → Distributor → Dealer, with sample `divisionId`s_

Each division stores both `divisionId` and `parentDivisionId`, so the tree is traversable in both directions — needed for dashboard data aggregation, where roll-ups require walking up the parent chain.

![Vertical chain of divisions showing each level's fields: divisionId and parentDivisionId](kia-division-fields-chain.png)
_Each level's fields: `divisionId` + `parentDivisionId` form the parent-pointer chain_

## Content Model

### Schema

In the `idcx-admin` client UI, a content looks like:

![DCE admin UI showing CARNIVAL content with General Settings panel and a sidebar listing child content types](dce-admin-ui-content.png){: .shadow .rounded-10 }
_DCE admin UI: one Content (e.g. `CARNIVAL`) with General Settings and a sidebar of its child content types_

A Content is the parent document; the sidebar lists its children:

- Docent Guide
- Story Contents
- Docent Tour — Exterior / Interior / Technology
- Promotion Video
- Spec Summary
- Dealer Specials
- QR Templates

Each child is its own document, linked back to the parent via `contentId`. The relationship is strict 1:1 — each Content has exactly one Guide, one Story, etc.

![Abstract schema: a Content parent (contentId, clusterId, status) with arrows to children Guide, Story, Exterior, Interior, all referencing back via contentId](content-children-schema.png)
_Abstract schema: the parent `Content` (with `contentId`, `clusterId`, `status`) and its 1:1 children that all reference back via `contentId`_

### Draft/Publish Lifecycle

#### Rules

- Only `DRAFT` status documents are directly editable.
- `PUBLISH` status documents:
  - are created by **copy** on first publish (a new PUBLISH doc gets a new `contentId`) or **overwrite** on re-publish (the existing PUBLISH doc is updated in place).
  - are visible only to **child** divisions — e.g. when a DISTRIBUTOR publishes a content, its DEALERs can see and add that content (the DISTRIBUTOR itself keeps editing the DRAFT).

The full state machine:

```mermaid
stateDiagram-v2
  [*] --> DRAFT : create
  DRAFT --> PUBLISH : publish (first time → copy creates new doc)
  PUBLISH --> PUBLISH : re-publish (overwrite)

  note right of PUBLISH
    When a child division adds a PUBLISH content,
    a new DRAFT is created at the child
    (new clusterId + contentId).
  end note
```

#### In Action

**Init.** DISTRIBUTOR `div_11` has two DRAFT contents (in clusters `clu_1` and `clu_2`). DEALER `div_111` has nothing — there's no PUBLISH content visible to it yet.

![Init state: DISTRIBUTOR div_11 has two DRAFT contents in clu_1 and clu_2; DEALER div_111 has no content](draft-publish-init.png)
_Init: DISTRIBUTOR has two DRAFT clusters; DEALER has no visible content_

**On Publish.** DISTRIBUTOR `div_11` publishes the content in `clu_1`. A new `con_2` PUBLISH document is created alongside the existing `con_1` DRAFT (1 DRAFT → 0..1 PUBLISH). The DEALER can now see `con_2` and add it.

![On publish: clu_1 now contains both con_1 (DRAFT) and con_2 (PUBLISH); arrow from con_2 to DEALER indicates visibility for add/update](draft-publish-on-publish.png)
_On publish: a new PUBLISH document is created in the same cluster; visible to DEALERs to add or update_

**On Add.** DEALER `div_111` adds the published content. A new `clusterId` (`clu_11`) and `contentId` (`con_11`) are generated for the DEALER's copy, and the new content starts in DRAFT status — the DEALER can edit it before publishing. (In this case, no further levels exist below DEALER, so the cycle terminates here.)

![On add: DEALER creates a new cluster clu_11 with a new content con_11 in DRAFT, copied from the parent's PUBLISH content](draft-publish-on-add.png)
_On add: DEALER creates a new cluster + content in DRAFT, copied from the parent's PUBLISH content_

## Takeaways

Three patterns are in play here, each chosen to fit a specific constraint:

- **Hierarchical tree with stored parent pointers** — explicit `parentDivisionId` makes upward roll-up (dashboard aggregation) cheap. An adjacency-list pattern, not nested-set or materialized-path; chosen because the tree is shallow (5 levels) and reads dominate writes.
- **Parent-children content with 1:1 typed children** — keeps each child's schema clean and independently evolvable, while the parent tracks aggregate state (`status`, `clusterId`). The UI's tabbed sidebar maps directly onto the children.
- **Draft/publish with copy semantics** — publishing creates a separate PUBLISH document rather than mutating the DRAFT in place. The editor keeps a working draft while subordinates see a stable snapshot. The cycle repeats per level: a parent's PUBLISH becomes the next level's DRAFT (under a new cluster/content id) once added.
