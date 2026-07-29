# Contribution 4: Add auto_updates true to Scapple cask

**Contribution Number:** 4

**Student:** Bhavya Agarwal

**Issue:** https://github.com/Homebrew/homebrew-cask/issues/170994 (scapple line item)

**Status:** Merged

---

## Why I Chose This Issue

Meta-issue #170994 tracks Casks using a `:sparkle`/`:electron_builder` livecheck strategy that lack `auto_updates true`. The `scapple` line item was unchecked with no PR linked, so it was unclaimed. Self-contained, low-risk change with a clear precedent from other merged PRs against the same meta-issue.

---

## Understanding the Issue

### Problem Description

`Casks/s/scapple.rb` uses `:sparkle` as its livecheck strategy but does not declare `auto_updates true`, even though the app has a working in-app Sparkle updater — meaning Homebrew doesn't need to reinstall the app on every version bump.

### Expected Behavior

If the app has a working in-app updater, the cask should declare `auto_updates true` so Homebrew knows the app updates itself outside of `brew upgrade`.

### Current Behavior

No `auto_updates` flag was present. Livecheck was already correct (cask version `1.5.4` matched the latest release in the Sparkle appcast at `literatureandlatte.com/downloads/scapple/scapple.xml`), so no version bump was needed.

### Affected Components

`Casks/s/scapple.rb`

---

## Reproduction Process

### Steps Taken

1. Fetched the appcast XML directly and confirmed `1.5.4` (the cask's declared version) matches the newest `<item>` entry — livecheck already correct.
2. Noted sibling cask `scrivener.rb` (same vendor, Literature & Latte, same S3/Sparkle infrastructure) already has `auto_updates true` — a signal, but not proof.
3. Per precedent from merged PR [Homebrew/homebrew-cask#205297](https://github.com/Homebrew/homebrew-cask/pull/205297) (5kplayer), maintainers expect the in-app updater to be directly observed, not just inferred.
4. Installed Scapple, opened the Scapple menu → "Check for Updates…" → Sparkle dialog appeared and reported no updates available, confirming the in-app updater works.

---

## Solution Approach

### Implementation

Added a single line, `auto_updates true`, to `Casks/s/scapple.rb` (matching the style/position used in `scrivener.rb` and the 5kplayer precedent PR).

---

## Pull Request

**PR Link:** https://github.com/Homebrew/homebrew-cask/pull/277363

**PR Description:** Documents the livecheck verification and in-app updater confirmation, references the `scapple` item in #170994 via "Addresses" (not an auto-close keyword).

**Maintainer Feedback:**
- 2026-07-26: Auto-closed by the `github-actions[bot]` "Check pull requests" workflow for an "incomplete or outdated pull request template". Root cause: the bot's `check_template.rb` requires the template's checkbox lines to match **verbatim** — I had substituted the `<cask>` placeholder with `scapple` and appended my own explanation onto the AI-disclosure checkbox line, which dropped the verbatim-match rate below its 75% threshold. Fixed by re-editing the PR body to keep every template line untouched (including the literal `<cask>` placeholder) and adding my actual verification notes as separate text alongside/below each item instead of editing the checklist lines themselves. The bot's `check-prs.yml` workflow then auto-reopened the PR on the next `edited` event.

- 2026-07-27: PR #277363 merged.

**Status:** Merged.

---

## Learnings & Reflections

### Technical Skills Gained

- Homebrew Cask's `auto_updates` semantics: it signals the app self-updates outside Homebrew's control, distinct from the `livecheck` block which just tracks the latest version for cask bumps.
- Sparkle appcast XML structure (`sparkle:shortVersionString`, `sparkle:version`) and how `:sparkle` livecheck strategy consumes it.

### Challenges Overcome

- Avoided the temptation to add `auto_updates true` purely by inference from the sibling `scrivener` cask — confirmed with an actual in-app check first, per established maintainer expectations on this meta-issue.

---

## Resources Used

- https://github.com/Homebrew/homebrew-cask/issues/170994
- https://github.com/Homebrew/homebrew-cask/pull/205297 (precedent: 5kplayer auto_updates)
- https://www.literatureandlatte.com/downloads/scapple/scapple.xml (Sparkle appcast)
