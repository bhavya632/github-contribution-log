# Contribution 3: Sort model list by tags via URL parameter

**Contribution Number:** 3

**Student:** Bhavya Agarwal

**Issue:** https://github.com/manyfold3d/manyfold/issues/4962

**Status:** Claimed, awaiting maintainer clarification on ranking logic

---

## Why I Chose This Issue

Labeled "good first issue" and "feature" in `manyfold3d/manyfold`, a child of the "XRForge wishlist" parent issue (#4960). Self-contained, user-facing feature work on a sort/query feature.

---

## Understanding the Issue

### Problem Description

The models page needs the ability to sort by one or more tags via a URL query parameter, e.g. `/models?sort=staffpicks,new,oldstaffpicks`. Unlike a filter, this is a sort: models should be ranked based on how many of the listed tags they have, not just included/excluded.

### Expected Behavior

Not yet finalized by the maintainer. Proposed interpretation (posted as a comment on 2026-07-26, awaiting reply from @Floppy): each model is ranked by how many of the listed tags it carries — a model matching all three tags in the example ranks above one matching two, and so on — with ties broken by the order tags are given in the URL (e.g. `staffpicks` outweighs `new`). Open question: whether models matching none of the listed tags should still appear (sorted last) or be excluded entirely.

### Current Behavior

No such sort capability currently exists.

### Affected Components

Likely the models controller/query layer handling the `/models` route and its `sort` param; exact files not yet explored (per user's choice, no codebase exploration has been done yet — waiting on maintainer confirmation of ranking semantics first).

---

## Solution Approach

### Analysis

Pending — implementation plan depends on the maintainer's answer to the ranking-logic question raised in the issue comment.

---

## Pull Request

**PR Link:** Not yet opened.

**Maintainer Feedback:**
- 2026-07-26: Comment posted asking @Floppy to confirm (a) ranking by tag-match count with order-based tie-breaking, and (b) whether zero-tag-match models are included at the bottom or excluded. No reply yet.

**Status:** Waiting on maintainer reply before starting implementation.

---

## Resources Used

- https://github.com/manyfold3d/manyfold/issues/4962
- https://github.com/manyfold3d/manyfold/issues/4960 (parent: XRForge wishlist)
