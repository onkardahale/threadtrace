---
name: case-study-financial-platform
description: Real case study showing threadtrace finding 6 confirmed bugs across 3 seeds in a production TypeScript/Firebase/Hasura financial data platform. Use as quality calibration — every threadtrace finding should match this standard of specificity, evidence, and concrete fixes.
---

# Case Study: Financial Data Platform — 6 Confirmed Bugs

threadtrace's first automated run on a production TypeScript/Firebase/Hasura codebase. A financial data platform ingesting market data, AI-generated analysis summaries, and fund management data. ~50K lines of TypeScript. No errors in logs.

## What threadtrace Found

6 confirmed bugs across 3 seeds. Zero false positives. Each finding independently verified by Minos with fresh context.

---

## Bug 1: Dead Retry Logic — .find() Missing Return Statement

**Seed**: "What happens when perform() and cleanUpJsonL() are called without await in AI batch retry?"
**Confidence**: 0.95 | **Severity**: CRITICAL | **Verified**: CONFIRMED

### The Thread

Icarus flagged the AI summary batch pipeline as highest damage potential. Theseus traced the retry path and found `.find()` with a block body `{}` but no `return` statement. Daedalus confirmed this is the only `.find()` in the codebase with this pattern — all siblings use arrow shorthand. Minos independently confirmed: the predicate always returns `undefined`, making lines 31-42 unreachable dead code.

### Evidence

```typescript
// retry.ts:25 — THE BUG
const resRequest = status.batch.requests.find((request)=>{request.id == resCustomId})
//                                                       ^ block body, no return
// .find() predicate always returns undefined
// .find() always returns undefined
// Line 26 check exits early
// Lines 31-42 (actual retry logic) are dead code
```

### Secondary Bug (same file)

```typescript
// retry.ts:50-54
removeLastId(request) {
  request.fileIds.slice(0, -1)  // .slice() is non-mutating, result never assigned
}
// This function is a no-op
```

### Impact

Every AI summary batch with partial failures permanently loses the failed records. No error, no log, no retry. The summaries silently disappear. Users relying on this analysis never know data is missing.

### Fix (1 line)

```diff
- const resRequest = status.batch.requests.find((request)=>{request.id == resCustomId})
+ const resRequest = status.batch.requests.find((request) => request.id === resCustomId)
```

---

## Bug 2: Cascading Unawaited Async — Race Condition Masked by Bug 1

**Seed**: Same as Bug 1
**Confidence**: 0.95 (latent) | **Severity**: HIGH | **Verified**: CONFIRMED

### The Thread

While tracing the retry path for Bug 1, Theseus documented three unawaited async calls. Daedalus found that all siblings in the codebase await their async operations — these are the only fire-and-forget calls. Minos confirmed the race condition and the masking relationship with Bug 1.

### Evidence

```typescript
// execute.ts:29 — not awaited
retry(openai, status, failedRecords)

// retry.ts:39 — not awaited
perform(openai, status.batch.fileIds, newRequests, status.article, status.batch.batchFileId)

// retry.ts:41 — not awaited
cleanUpJsonL(openai, status.batch.batchId)
```

### The Chain

This bug is currently masked by Bug 1 (the dead `.find()` means retry never executes). If someone fixes Bug 1 without fixing these:
1. `retry()` fires as a detached promise, races against `cleanup()` in the caller
2. Both write `.set()` to same Firestore doc — last-write-wins corruption
3. `cleanup()` deletes OpenAI files that `retry()`'s `perform()` still references
4. Firebase function instance may terminate before detached promises resolve

### Fix (3 lines)

```diff
- retry(openai, status, failedRecords)
+ await retry(openai, status, failedRecords)

- perform(openai, status.batch.fileIds, newRequests, status.article, status.batch.batchFileId)
+ await perform(openai, status.batch.fileIds, newRequests, status.article, status.batch.batchFileId)

- cleanUpJsonL(openai, status.batch.batchId)
+ await cleanUpJsonL(openai, status.batch.batchId)
```

---

## Bug 3: User Creation — Non-Atomic, Non-Idempotent, No Rollback

**Seed**: "What happens when setCustomUserClaims fails after insertUserOne succeeds in user creation?"
**Confidence**: 0.90 | **Severity**: HIGH | **Verified**: CONFIRMED

### The Thread

Icarus identified user creation as a Tier 2 (auth) risk. Theseus traced the three-step sequence: insert user → set JWT claims → send notification. Daedalus found no rollback/saga/compensation patterns anywhere in the codebase. Minos confirmed: Firebase v1 `auth.user().onCreate` does NOT retry on failure, and the insert mutation has no `on_conflict` clause.

### Evidence

```typescript
// perform.ts:11-13
const { id, isStaff } = await insertUserOne({ uid, email, displayName })  // committed to Hasura
await saveClaims({ uid, id, email, isStaff })                              // may fail — no rollback
await sendNotification({ id, email })                                        // never reached if above fails
```

Three problems compound:
1. No try/catch, no rollback of the Hasura insert
2. Firebase v1 auth trigger does NOT retry — failure is permanent
3. `insert_users_one` has no `on_conflict` — manual replay fails with unique constraint violation
4. User is stuck: exists in DB, has no JWT claims, can't access any data

### Impact

User authenticates to Firebase successfully but is locked out of all Hasura data. Requires manual database intervention to fix. No monitoring detects this state.

### Fix

```typescript
// Add on_conflict to mutation
insert_users_one(object: $user, on_conflict: { constraint: users_uid_key, update_columns: [email, displayName] })

// Wrap in retry-safe logic or migrate to Cloud Task (v2, retryable)
```

---

## Bug 4: setClient Claims Overwrite — Landmine in Commented Code

**Seed**: Same as Bug 3
**Confidence**: 0.85 | **Severity**: MEDIUM (not deployed) | **Verified**: CONFIRMED

### The Thread

While tracing auth patterns for Bug 3, Theseus found a commented-out Firestore trigger. Daedalus checked git history — it was disabled but never removed. Minos confirmed: if re-enabled, `setCustomUserClaims` OVERWRITES all claims (does not merge), destroying the Hasura claims set by the user creation flow.

### Evidence

```typescript
// setClient.ts:40-41
await doc.set(data)                                                        // committed
await auth.setCustomUserClaims(uid, { clients: { [doc.id]: 'admin' } })    // OVERWRITES all claims
```

The `onCreateEvent` wrapper swallows errors and writes `{ status: 400 }` to a document that nothing reads.

### Impact

Currently mitigated (trigger commented out at index.ts:357-366). If re-enabled: client document persists but user can't access it, plus all existing Hasura claims destroyed for that user.

---

## Bug 5: GraphQL Field Errors Silently Dropped — 26 Loaders Affected

**Seed**: "Do GraphQL mutations silently succeed when Hasura returns errors alongside data?"
**Confidence**: 0.65 | **Severity**: HIGH | **Verified**: CONFIRMED

### The Thread

Icarus flagged the GraphQL layer as a risk. Theseus traced the `loadData()` utility used by 26 loaders and found it returns `data` even when `error` is present — urql's `CombinedError` with `graphQLErrors` is silently ignored. Only `networkError` is checked. Minos confirmed: field-level GraphQL errors (validation failures, permission errors, partial data) pass through undetected.

### Evidence

```typescript
// loadData returns data without checking error
const { data, error } = await client.query(...)
if (error?.networkError) throw error  // only network errors caught
return data                            // graphQLErrors silently ignored
```

### Impact

26 loaders serve stale or incorrect data when Hasura returns field-level errors. No monitoring detects this — the response has HTTP 200 and `data` is present.

---

## Bug 6: Queue Worker Permanent Hang on Unexpected Exceptions

**Seed**: Same as Bug 5
**Confidence**: 0.70 | **Severity**: MEDIUM | **Verified**: CONFIRMED

### The Thread

While tracing error paths through the batch processing pipeline, Theseus found that `pushAsync()` from the `async` library returns a Promise that never settles if the worker callback isn't called. Unhandled exceptions in the worker bypass the callback entirely. Minos confirmed: the queue permanently stalls with no timeout, no error log, and no recovery.

### Evidence

```typescript
// Worker function — callback never called on exception
queue.push(task, (err) => { /* never reached if worker throws */ })
// pushAsync() wraps this — Promise never settles
// Queue permanently stalled
```

### Impact

Any unexpected exception in the worker function permanently stalls the queue. No timeout mechanism, no dead letter queue, no monitoring.

---

## What threadtrace Did Differently

1. **Bug chaining**: Bug 1 (dead retry) → led to discovering Bug 2 (unawaited async) was masked by Bug 1. Fixing one without the other makes things worse.

2. **Read library source**: Minos verified Firebase v1 auth triggers don't retry by reading the Firebase SDK source. This is undocumented behavior that makes Bug 3 permanent.

3. **Found the landmine**: Bug 4 is in commented-out code. A future developer re-enabling it would silently destroy user auth state. No scanner flags commented code.

4. **Scale detection**: Bug 5 affects 26 loaders sharing the same error suppression pattern. threadtrace traced the shared utility, not each individual loader.

5. **Zero false positives**: All 6 findings confirmed by independent verification. Minos tried to disprove each one and couldn't.

## Patterns Learned

- No `@typescript-eslint/no-floating-promises` ESLint rule — unawaited async calls undetected
- Zero test coverage for `aiSummariesV2` module
- No rollback/saga/compensation patterns anywhere in the codebase
- `setCustomUserClaims` overwrites (does not merge) — Firebase SDK behavior verified in source
- `onCreateEvent` wrapper swallows all errors — writes status docs nothing reads
- 26 loaders share same `loadData()` error suppression — only `networkError` checked
- `async` library `pushAsync()` Promise never settles if callback not called
