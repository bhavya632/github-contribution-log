# github-contribution-log
# Contribution 1: Fix async drop-event log throttling race in AsyncQueueListener

**Contribution Number:** 1

**Student:** Bhavya Agarwal

**Issue:** https://github.com/apache/gravitino/issues/10169

**Status:** Phase I - In Progress

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
