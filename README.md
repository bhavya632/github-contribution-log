# github-contribution-log
# Contribution 1: Fix async drop-event log throttling race in AsyncQueueListener

**Contribution Number:** 1

**Student:** Bhavya Agarwal

**Issue:** https://github.com/apache/gravitino/issues/10169

**Status:** Phase III - In Progress

---

## Why I Chose This Issue

The description is really good. The maintainer literally tells you what to do. There is only one comment, no open PR's, no assignee and matches my skill and known languages. 

---

## Understanding the Issue

### Problem Description

`AsyncQueueListener` drops events when its internal queue is full, and calls `logDropEventsIfNecessary()` to log a warning at most once every 60 seconds so a burst of drops doesn't flood the logs. The drop *count* (`dropEventCounters` / `lastDropEventCounters`) is tracked with `AtomicLong`s, but the throttling timestamp (`lastRecordDropEventTime`) is a plain `Instant` field with no synchronization or `volatile` keyword. When multiple threads drop events concurrently, they can each read a stale value of `lastRecordDropEventTime`, so the "has 60 seconds passed?" check and the counter update aren't treated as one atomic operation.

### Expected Behavior

At most one thread should log a drop-event warning per 60-second window, and the count reported in that warning should account for all drops that occurred since the last logged warning.

### Current Behavior

Because the time-gate check (`Instant.now().isAfter(lastRecordDropEventTime.plusSeconds(60))`) and the `lastDropEventCounters` CAS + `lastRecordDropEventTime` write aren't guarded together, two race conditions are possible:
1. Two threads can both pass the time-gate check before either updates `lastRecordDropEventTime`, so both proceed toward logging in the same window (only one wins the CAS on `lastDropEventCounters`, but the intent of "log once per 60s" can still be violated depending on interleaving).
2. Without `volatile`, a thread on one CPU core may not see another thread's write to `lastRecordDropEventTime`, so it can keep evaluating against a stale timestamp — either causing duplicate warnings inside a window, or (less likely) suppressing a warning that should have fired.

### Affected Components

- `core/src/main/java/org/apache/gravitino/listener/AsyncQueueListener.java` — specifically `logDropEventsIfNecessary()` (the racy method) and `enqueueEvent()` (its only caller, invoked from `onPreEvent`/`onPostEvent`, which can run on many concurrent producer threads).

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

Root cause: `logDropEventsIfNecessary()` treats "check the time gate" and "update the counter + timestamp" as two separate, unsynchronized steps, even though the counter half (`lastDropEventCounters`) is atomic:

```java
private void logDropEventsIfNecessary() {
  long currentDropEvents = dropEventCounters.incrementAndGet();
  long lastDropEvents = lastDropEventCounters.get();
  long dropEvents = currentDropEvents - lastDropEvents;
  if (dropEvents > 0 && Instant.now().isAfter(lastRecordDropEventTime.plusSeconds(60))) {
    if (lastDropEventCounters.compareAndSet(lastDropEvents, currentDropEvents)) {
      LOG.warn(...);
      lastRecordDropEventTime = Instant.now();
    }
  }
}
```

`lastRecordDropEventTime` is a plain field, so (a) it has no happens-before guarantee across threads (no `volatile`, no lock), and (b) the read of it and the later write to it aren't part of the same critical section as the `compareAndSet`. The CAS on `lastDropEventCounters` prevents two threads from double-counting the *same* drop range, but it does nothing to serialize the time-gate check itself, so the "once per 60s" guarantee isn't actually enforced under concurrent drops.

### Proposed Solution

Make the timestamp read/check/write part of the same synchronized unit as the counter CAS, rather than fixing the count and the time gate independently. Two viable options, in increasing order of robustness:

1. **Minimal fix:** mark `lastRecordDropEventTime` `volatile` so writes are visible across threads. This fixes the stale-read/visibility half of the bug but still leaves a narrow window where two threads can both pass the time-gate check before either writes the new timestamp.
2. **Stronger fix (preferred):** wrap the check-CAS-update block in a `synchronized` block (or replace `lastRecordDropEventTime`/`lastDropEventCounters` with a single `AtomicReference<DropState>` holding both the count and timestamp, updated via one `compareAndSet`) so the whole "has 60s passed? if so, claim this window and update state" sequence is a single atomic critical section, matching what the issue asks for.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `logDropEventsIfNecessary()` in `AsyncQueueListener` is supposed to log a drop-event warning at most once per 60-second window. The counter (`dropEventCounters`/`lastDropEventCounters`) is atomic, but the throttling timestamp (`lastRecordDropEventTime`, a plain `Instant`) is not, so the "has 60s passed?" check and the "claim this window" update aren't one atomic operation. Under concurrent drops this can produce duplicate warnings in the same window or hide a warning that should have fired.

**Match:** There's no existing test file for `AsyncQueueListener` (`core/src/test/java/org/apache/gravitino/listener/` only has `TestEventListenerManager.java`), so there's no prior pattern in this class to copy directly. The closest precedent in the module is `TestEventListenerManager`'s use of `DummyEventListener`/`DummyAsyncEventListener` test doubles and JUnit 5 (`org.junit.jupiter.api`) — a new `TestAsyncQueueListener` should follow that same style and package. For the fix itself, the project already relies on `java.util.concurrent.atomic` types elsewhere in this file (`AtomicBoolean`, `AtomicLong`), so replacing the split `Instant` + `AtomicLong` state with a single `AtomicReference` holding both values keeps the fix idiomatic to the surrounding code rather than introducing a new locking style.

**Plan:**
1. In `core/src/main/java/org/apache/gravitino/listener/AsyncQueueListener.java`, replace the separate `lastDropEventCounters` (`AtomicLong`) and `lastRecordDropEventTime` (`Instant`) fields with a single immutable state object (e.g. `AtomicReference<DropWindowState>` holding `{count, windowStart}`), so the time-gate check and the update happen inside one `compareAndSet`/CAS loop.
2. Update `logDropEventsIfNecessary()` to read the combined state once, evaluate the 60-second gate against it, and attempt to CAS in the new state (new timestamp + counter) — only the thread that wins the CAS logs the warning, closing the two-thread race described in Analysis.
3. Add `core/src/test/java/org/apache/gravitino/listener/TestAsyncQueueListener.java` with a concurrency test: spin up several threads calling `enqueueEvent()` against a listener with a full queue, and assert the warning is logged at most once per 60-second window (using an injectable clock or a shortened window for the test) and that the reported drop count matches the number of drops since the last log.
4. Run the existing `core` module test suite (`./gradlew :core:test`) to confirm no regressions in `TestEventListenerManager` or other listener tests.

**Implement:** [Link to your branch/commits as you work]

**Review:** Before opening the PR, confirm: the fix keeps `logDropEventsIfNecessary()` lock-free/wait-free consistent with the rest of the class (no blocking under the hot event path), the new test is deterministic (no reliance on real 60-second sleeps), the diff follows Gravitino's Apache license header and checkstyle conventions, and the PR references issue #10169 and summarizes the race in plain terms for the maintainer.

**Evaluate:** Verify with the new concurrency unit test (run repeatedly, e.g. `--tests TestAsyncQueueListener -Djunit.jupiter.execution.parallel.enabled=true` or a loop of N runs, to catch flakiness from the original race), plus a manual local run that floods the queue from multiple threads and inspects the logs for exactly one warning per window.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]

---
---

# Contribution 2: Clarify namesrv.toml vs namesrv-example.toml and fix placeholder config value

**Contribution Number:** 2

**Student:** Bhavya Agarwal

**Issue:** https://github.com/mxsm/rocketmq-rust/issues/7622

**Status:** Phase IV - PR opened, awaiting maintainer review (PR #8396)

---

## Why I Chose This Issue

Labeled "good first issue" / "Difficulty level/Easy", scoped to documentation, and the maintainer (`mxsm`) had already left concrete guidance in a comment (units/defaults/expected ranges, a small-dev vs. production baseline pair, and a possible config-parser fixture test). The maintainer had tagged another user (`@hiSandog`) two weeks prior with no visible follow-up, so I commented on the issue first to check it was still open before starting, and proposed a plan matching the maintainer's comment. I also flagged upfront that I don't speak Chinese, so `README-zh_cn.md` would need to stay a follow-up rather than something I fake through translation — waiting on the maintainer's reply before writing any code.

---

## Understanding the Issue

### Problem Description

`rocketmq-namesrv` ships two config files: `resource/namesrv.toml` and `resource/namesrv-example.toml`. The README's Quick Start section tells users to start the server with `-c rocketmq-namesrv/resource/namesrv-example.toml` (the fully annotated file), but doesn't explain what `namesrv.toml` is for. As it stands, `namesrv.toml` contains nothing but:

```toml
rocketmqHome=11111
```

A numeric value assigned to a path-typed field, with no other keys, no comments, and no indication it's a placeholder rather than a working config. A first-time user who opens `namesrv.toml` expecting a usable example (reasonable, given `namesrv-example.toml` sits right next to it and *is* fully documented) gets nothing useful and no signal that they're looking at the wrong file.

### Expected Behavior

- The README (both English and, eventually, Chinese) should state plainly why two files exist and which one to use for what purpose.
- `resource/namesrv.toml` should either contain a real, documented minimal example (per the maintainer's "small dev / production baseline" suggestion) or be unambiguously marked as a non-functional placeholder — not silently ship a garbage value that looks like an oversight.
- Quick Start commands should be double-checked to confirm they reference the file that's actually meant to be run.

### Current Behavior

- `rocketmq-namesrv/README.md`'s Quick Start section already points at `namesrv-example.toml` for the "start from a config file" case, and the Configuration section's key table also links to `namesrv-example.toml` — so the *commands* are pointed correctly today. The gap is that nothing explains what `namesrv.toml` is, so it reads as an inconsistency rather than an intentional placeholder.
- `resource/namesrv.toml` is exactly one line (`rocketmqHome=11111`) plus the license header — not a valid directory path, not documented, not aligned with any of the "small dev" or "production baseline" framing the maintainer asked for.
- `resource/namesrv-example.toml` is well documented (comments with defaults, units where relevant, and env var alternatives for `rocketmqHome`) but is a single "documents every key" file, not a curated dev/prod pair.
- `README-zh_cn.md` mirrors the English structure closely today, so whatever wording is added to the English README needs a Chinese equivalent to stay aligned — the piece I can't do myself.

### Affected Components

- `rocketmq-namesrv/README.md` — Quick Start / Configuration sections (needs the two-file explanation).
- `rocketmq-namesrv/README-zh_cn.md` — same sections, Chinese (blocked on translation help).
- `rocketmq-namesrv/resource/namesrv.toml` — the placeholder file itself.
- `rocketmq-namesrv/resource/namesrv-example.toml` — possibly split or supplemented with a "production baseline" variant per the maintainer's comment.
- Config parsing (`NamesrvConfig`, referenced from `src/bin/namesrv_bootstrap_server.rs`) — only in scope if the maintainer wants the "docs test / parser fixture" idea included in this PR rather than deferred.

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. Open `rocketmq-namesrv/resource/namesrv.toml` expecting a runnable minimal config, per the README's framing of it alongside `namesrv-example.toml`.
2. Observe it contains only `rocketmqHome=11111` — not a valid path, not documented, no indication it's intentional.
3. Compare against `namesrv-example.toml`, which is fully annotated, and note the README doesn't explain the difference.

### Reproduction Evidence

No reproduction commit/screenshots needed — this was a documentation and example-config issue (a broken placeholder file and a missing README explanation), not a code bug with runtime behavior to capture. The "steps to reproduce" above (opening `namesrv.toml` and comparing it to `namesrv-example.toml`) are directly observable by reading the files in the repo, and are what verifying the fix requires.

---

## Solution Approach

### Analysis

This is pending maintainer confirmation on scope (see open questions in my issue comment). The likely shape of the fix, based on `mxsm`'s guidance:
1. `namesrv.toml` becomes either a small, documented "dev baseline" (minimal keys, real defaults, units/ranges noted) or is explicitly labeled a placeholder/template in both its own comments and the README.
2. `namesrv-example.toml` stays (or becomes) the fully annotated reference, potentially supplemented with a "production baseline" variant.
3. Both READMEs gain a short paragraph explaining the two-files convention, kept aligned English/Chinese.
4. A config-parser fixture/docs test (asserting the documented keys match `NamesrvConfig`'s actual fields) may or may not land in this same PR — open question to the maintainer.

### Proposed Solution

Maintainer (`mxsm`) responded on the issue: *"Don't worry about the Chinese text for now. Just submit the PR."* — this confirms an English-only PR is acceptable, with the Chinese README sync deferred as a follow-up. The fixture-test question wasn't addressed explicitly, but I found `rocketmq-namesrv/src/namesrv_config_parse.rs` already has a `namesrv_config_parse_loads_example_file` test that parses `namesrv-example.toml` through `NamesrvConfig` — so extending that same pattern to cover the new/edited files (rather than adding a separate standalone fixture test) satisfies the spirit of the ask with minimal new surface area.

Final shape of the change:
1. `resource/namesrv.toml` becomes a small, documented "dev baseline" — a handful of commonly-changed keys (paths under `/tmp` for a self-contained local run) with comments, rather than the `rocketmqHome=11111` placeholder.
2. New `resource/namesrv-production.toml` — a "production baseline" with thread-pool/queue sizes tuned above the built-in defaults, each documented with its default value and a suggested range, per the maintainer's original ask.
3. `namesrv-example.toml` stays as-is: the exhaustive, fully-annotated reference of every key (already well documented, per Analysis above).
4. `README.md`'s Quick Start and Configuration sections gain a short explanation of what each of the three files is for, and the Quick Start commands are re-verified against the edited files.
5. `namesrv_config_parse.rs` gains tests parsing the new dev/production files through `NamesrvConfig`, so drift between the docs and the parser fails a test instead of silently landing.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Two namesrv config files exist for different purposes, but only one is documented, and the placeholder (`namesrv.toml`) is indistinguishable from a broken example. Fix is scoped to docs + one config file, with an open question on whether Chinese docs and a parser fixture test are in scope for this PR.

**Match:** No existing "two-tier config example" pattern elsewhere in the workspace to copy from directly; `namesrv-example.toml`'s existing comment style (default value + short purpose line, occasionally an env var alternative) is the style to extend to any new/edited file.

**Plan:**
1. Get scope confirmation from `mxsm` on the Chinese README and parser-fixture questions.
2. Update `rocketmq-namesrv/README.md`'s Quick Start/Configuration sections to explain the `namesrv.toml` vs. `namesrv-example.toml` distinction.
3. Replace the `rocketmqHome=11111` placeholder in `resource/namesrv.toml` with either a documented minimal dev config or an explicit "this is a placeholder, see namesrv-example.toml" comment block.
4. Re-verify every Quick Start command still targets the intended file after the change.
5. Mirror the README wording change into `README-zh_cn.md` (myself if scope allows using careful, verified translation tooling and a native-speaker review request, or hand off, per maintainer's answer).
6. If in scope: add a docs test / parser fixture asserting the keys documented in `namesrv-example.toml` match `NamesrvConfig`'s actual serde field names.

**Implement:** [Link to your branch/commits as you work]

**Review:** Before opening the PR, confirm: English and Chinese READMEs (if both included) say the same thing, `namesrv.toml` no longer looks like an unintentional bug, Quick Start commands were actually re-run against the edited files, and any added fixture test fails if the docs and parser drift apart.

**Evaluate:** Manually run the Quick Start commands from the README against both config files post-edit; if a fixture test is added, confirm it fails when a key is deliberately renamed in one file but not the other.

---

## Testing Strategy

### Unit Tests

All added in `rocketmq-namesrv/src/namesrv_config_parse.rs`, extending the module's existing `#[cfg(test)]` block (which already had a fixture test for `namesrv-example.toml`) rather than introducing a new test file:

- [x] `namesrv_config_parse_loads_dev_baseline_file` — loads `resource/namesrv.toml` (the rewritten dev baseline) through `parse_command_and_config_file` and asserts `rocketmq_home`, `kv_config_path`, and `config_store_path` come back as the `/tmp/rocketmq` values set in the file, i.e. the file actually parses and isn't silently falling back to defaults.
- [x] `namesrv_config_parse_loads_production_baseline_file` — loads the new `resource/namesrv-production.toml` and asserts `rocketmq_home` plus every thread-pool/queue-capacity value (`client_request_thread_pool_nums`, `default_thread_pool_nums`, `client_request_thread_pool_queue_capacity`, `default_thread_pool_queue_capacity`, `unregister_broker_queue_capacity`) and `use_route_info_manager_v2` match what the file declares — the whole point of the "production baseline" is those tuned numbers, so the test pins them.
- [x] Pre-existing, unchanged: `namesrv_config_parse_reads_selected_toml_fields`, `namesrv_config_parse_loads_example_file`, `namesrv_config_parse_falls_back_to_default_for_missing_file` — all still pass, confirming the edits didn't regress the example file or the default-fallback path.

Together these three fixture tests (existing example-file test + 2 new ones) are the "config parser fixture" the maintainer asked for in the issue thread: if a key gets renamed in `NamesrvConfig` (or a typo lands in one of the `.toml` files) without the other side being updated, one of these tests fails instead of the drift going unnoticed.

### Integration Tests

- [x] Ran the actual `rocketmq-namesrv-rust` binary against all three config files (`namesrv.toml`, `namesrv-example.toml`, `namesrv-production.toml`) using `-c <file> -p` (print merged config and exit), confirming the Quick Start commands documented in the README work end-to-end against the real CLI, not just the parser in isolation.

### Manual Testing

- `cargo fmt -p rocketmq-namesrv` — no changes needed.
- `cargo clippy -p rocketmq-namesrv --all-targets --all-features -- -D warnings` — clean.
- `cargo test -p rocketmq-namesrv --lib namesrv_config_parse` — 5/5 passed (the 3 pre-existing + 2 new tests above).

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:**
  - `rocketmq-namesrv/resource/namesrv.toml` (rewrote placeholder into a documented dev baseline)
  - `rocketmq-namesrv/resource/namesrv-production.toml` (new)
  - `rocketmq-namesrv/README.md` (Quick Start + Configuration sections)
  - `rocketmq-namesrv/src/namesrv_config_parse.rs` (two new fixture tests)
- **Key commits:** `dbb8e1a6` / `0d72ec69` (author identity amended) on branch `fix-7622-namesrv-toml-docs`, single commit `[ISSUE #7622]📝Clarify namesrv.toml vs namesrv-example.toml and add production baseline`
- **Approach decisions:**
  - Kept `namesrv-example.toml` untouched — it already serves as the "documents every key" reference; only the broken/undocumented `namesrv.toml` needed fixing, plus a new production-tier file.
  - Included the config-parser fixture tests even though the maintainer's "just submit the PR" reply didn't explicitly confirm that part of scope, because it was directly requested in the maintainer's original comment and reused an existing test pattern already in `namesrv_config_parse.rs` (low cost, high alignment with what was asked).
  - Had no local Rust toolchain on this machine at the start of this session — installed `rustup` (auto-synced to the `nightly` channel pinned by `rust-toolchain.toml`) and Homebrew's `protobuf` (needed by a transitive build script for `rocketmq-controller`) before validation could run.

---

## Pull Request

**PR Link:** https://github.com/mxsm/rocketmq-rust/pull/8396

**PR Description:** Used "Addresses #7622" rather than "Closes #7622" in the PR body, intentionally, so the issue isn't auto-closed by GitHub until the maintainer has actually reviewed and merged.

**Maintainer Feedback:**
- 2026-07-15 (approx.): Posted a comment on the issue confirming it's unassigned and proposing a plan aligned with `mxsm`'s guidance; asked whether an English-only PR (Chinese README as follow-up) is acceptable, and whether the config-parser fixture belongs in this PR or a later one.
- 2026-07-19: `mxsm` replied: "Don't worry about the Chinese text for now. Just submit the PR." — scope unblocked, Chinese README deferred, proceeding with English-only PR.
- 2026-07-19: Opened PR #8396 with the dev/production baseline configs, README updates, and extended parser fixture tests.

**Status:** Awaiting maintainer review

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- Issue thread: https://github.com/mxsm/rocketmq-rust/issues/7622
- `rocketmq-namesrv/README.md` and `README-zh_cn.md` (current Quick Start / Configuration sections)
- `rocketmq-namesrv/resource/namesrv.toml` and `resource/namesrv-example.toml`
