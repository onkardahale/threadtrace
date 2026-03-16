# Methodology

threadtrace finds correctness bugs — code that runs without errors but produces wrong results. Silent data loss, swallowed exceptions, dead error paths, masked race conditions.

This document explains why it works.

## The Core Problem

Most bug-finding tools check syntax, types, or known vulnerability patterns. They operate on the code in front of them.

Correctness bugs live in the **gaps between** files, libraries, and assumptions. A `.find()` with a block body and no `return`. A retry path that's been dead code since the day it was written. An `await` that was never added. These pass every linter, every type checker, every test you wrote — because you didn't know to test for them.

Finding these bugs requires tracing what actually happens when something fails, across file boundaries, into library source code, through the full error path until it reaches a terminal state.

## Architecture

Four agents with independent context. The agent that discovers a potential bug is never the agent that verifies it.

### Phase 0: Reconnaissance (Icarus — haiku)

A fast, cheap scout scans the codebase and builds a map:
- What are the critical operations? (database writes, payments, auth flows)
- Where are the external boundaries? (APIs, libraries, cloud services)
- What are the error handling hotspots?

Icarus ranks everything by damage potential: "if this fails silently, how bad is it?" The highest-damage operation becomes the first investigation seed.

Icarus maps. It never investigates.

### Phase 1: Thread Tracing (Theseus)

Theseus receives a seed question like *"What happens when a batch retry fails?"* and traces the error path end-to-end:

1. Find the entry point
2. Follow every branch, catch, callback, and return
3. Cross file boundaries — read the callers and callees
4. Read library source in `node_modules/` — not documentation, actual code
5. Continue until reaching a terminal state: error surfaced, logged and retried, swallowed, or unhandled

**Theseus is neutral.** It documents behavior without judging it. "This function catches the error and returns an empty array" — not "This function incorrectly swallows the error." Premature classification biases everything downstream.

### Phase 2: Pattern Comparison (Daedalus — sonnet)

Daedalus takes the trace and asks: how do the siblings handle this?

- Find 3-5 functions that do the same kind of operation
- Compare error handling across all of them
- Check git history — was this file recently changed? Is there a partial fix?

The output is a comparison table, not a verdict. "4 out of 5 re-throw. This one swallows. The sibling was patched 2 weeks ago." That's a data point. The orchestrator decides what it means.

### Phase 3: Hypothesis Formation (Orchestrator)

The orchestrator synthesizes the trace and pattern comparison into structured hypotheses:

- **Mechanism**: What specifically happens (code path + behavior)
- **Downstream effect**: What breaks as a result
- **Blast radius**: How much of the system is affected
- **Confidence score**: 0.0–1.0 based on evidence strength
- **Concrete fix**: 1–3 lines that would resolve it

This phase is never delegated. The orchestrator has full context from both Theseus and Daedalus and forms the hypothesis itself.

### Phase 4: Independent Verification (Minos)

This is where most bug-finding approaches fail. The person who found the bug is the person who evaluates it. Confirmation bias is structural.

Minos receives:
- The hypothesis, neutrally framed
- The file paths to investigate
- Nothing else

Minos never sees:
- How the hypothesis was formed
- The confidence score
- The evidence from Theseus or Daedalus
- Whether anyone thinks this is actually a bug

Minos's default stance is that the code is correct. It actively searches for reasons the hypothesis is wrong:
- Is there upstream handling that catches this?
- Is this intentional design (e.g., fire-and-forget by choice)?
- Is there a retry mechanism elsewhere?
- Does monitoring cover this case?

If Minos can't disprove it, it writes a concrete test case demonstrating the bug. If it can disprove it, it explains why.

### Phase 5: Report & Continue

Confirmed findings are published with full evidence chains. The orchestrator derives new seeds from what was found — a dead retry path might reveal unawaited async calls, which might reveal a broader pattern across the codebase.

Investigation continues automatically across seeds. State persists to disk so it survives session boundaries.

## Why Context Isolation Matters

The standard approach: one agent finds something suspicious, reasons about it, and decides if it's a real bug. This has a structural flaw — the agent that formed the hypothesis is motivated to confirm it. It's already primed with the evidence, the narrative, the "aha" moment.

threadtrace eliminates this by making the verifier (Minos) truly independent:

- Separate agent instance with no shared memory
- Receives only the hypothesis and file paths
- Never sees discovery reasoning or confidence scores
- Default stance is "the code is correct"

This isn't a prompt trick ("be skeptical"). It's architectural — Minos literally cannot access the discovery context because it was never provided.

The result: zero false positives across every run so far. When Minos confirms a bug, it's because the code is actually broken, not because it was primed to think so.

## Why Read Library Source

Most tools treat library calls as black boxes. They check if you're calling the API correctly based on documentation and type signatures.

But libraries have behaviors that aren't in the documentation:
- `setCustomUserClaims` in Firebase Auth **overwrites** all claims, it doesn't merge
- urql returns `data` even when `error` is present — they're independent fields
- `async` library's `pushAsync` returns a Promise that never settles if the worker callback isn't called

threadtrace reads the actual library source in `node_modules/`. When Theseus traces into a library call, it opens the file and reads what the function actually does. When Minos verifies, it checks the library version in `package.json` and reads the implementation.

Documentation is a summary. Source code is ground truth.

## Seed Selection

Investigation quality depends on starting with the right question. threadtrace organizes seeds by damage tier:

1. **Financial/Data Integrity** — database writes, payments, queue processing
2. **Authentication/Authorization** — auth flows, permission checks, token handling
3. **External Services** — API clients, notifications, file storage
4. **Async Patterns** — Promise handling, event flows, callbacks
5. **Data Transformation** — type coercion, validation, serialization

Within each tier, seeds are customized to the specific codebase based on Icarus's reconnaissance. Generic questions like "what happens when the database fails?" become specific ones like "what happens when `insertUserOne` fails after `setCustomUserClaims` succeeds in the user creation trigger?"

The best seeds come from domain understanding, not templates.

## Bug Chains

The most valuable findings aren't individual bugs — they're chains where bugs interact.

In the first production run, threadtrace found a `.find()` with a missing `return` statement that made the entire retry path dead code. While tracing that path, it discovered three unawaited async calls that create a race condition — currently invisible because the dead retry prevents them from executing.

Fixing the `.find()` without fixing the `await`s would activate the race condition. The bugs are linked. Neither is visible in isolation.

This is why threadtrace continues investigating after finding a bug instead of stopping. Each finding generates new seeds, and the investigation builds on itself.

## Design Principles

The architecture isn't arbitrary. Each constraint exists because removing it produces false positives, missed bugs, or wasted investigation.

### Separate discovery from judgment

When one agent finds something suspicious and then evaluates whether it's real, confirmation bias is structural — not a personality flaw you can prompt away. The agent has already built a narrative. It's primed.

threadtrace makes this impossible architecturally. Minos has no project memory, receives no discovery context, and starts from "the code is correct." This isn't "be skeptical" in a system prompt. It's a separate agent instance that literally cannot access the discovery reasoning.

### Observe before you classify

Theseus documents behavior neutrally: "this function catches the error and returns an empty array." Not "this function incorrectly swallows the error."

Why: premature classification corrupts downstream reasoning. If Theseus says "bug," Daedalus looks for confirming patterns instead of comparing objectively. If the trace says "behavior," Daedalus compares siblings on equal footing — and the deviation becomes evidence, not assumption.

The orchestrator classifies. Nobody else does.

### Centralize synthesis, distribute collection

Icarus collects the map. Theseus collects the trace. Daedalus collects the comparisons. But hypothesis formation stays with the orchestrator — always.

Why: the orchestrator is the only agent with full context from every source. Delegating hypothesis formation to a subagent means that agent would need the trace AND the comparisons AND the map, which means either duplicating context (expensive) or compressing it (lossy). The orchestrator already has it.

### Maximize damage, not coverage

Icarus ranks by "if this fails silently, how bad is it?" — not "how likely is this to have a bug?" or "how complex is this code?"

A low-probability bug in the payment path matters more than a high-probability bug in a logging utility. Seeds are ordered by blast radius because investigation time is finite and the goal is finding bugs that actually hurt.

### Trace to terminal state

Theseus doesn't stop at "the error is caught." It continues: caught by what? Re-thrown? Logged? Swallowed? Returned as a default value? Passed to a callback that nobody checks?

Most error handling bugs aren't at the throw site — they're three handlers deep, where someone assumed the caller would deal with it and the caller assumed the callee already did.

### Read source, not docs

Libraries have undocumented behaviors that create correctness bugs in calling code. `setCustomUserClaims` overwrites instead of merging. urql returns `data` alongside `error`. `pushAsync` never settles if the callback isn't called.

These aren't edge cases — they're the normal behavior of widely-used libraries that happens to differ from what the documentation implies. The only way to catch bugs caused by these behaviors is to read the actual implementation.

### Continue after finding

Most tools stop after finding a bug. threadtrace continues because bugs interact.

A dead retry path masks a race condition. Fixing the retry without fixing the race makes production worse. A missing `await` is invisible until the error path it guards becomes reachable.

Each finding generates new seeds. The investigation compounds — patterns learned from seed 1 inform seed 2, and false positive signatures from earlier runs prevent wasted verification in later ones.

### Persist everything

Investigation state (seeds completed, findings, patterns learned, false positive signatures) writes to disk after every phase. This serves two purposes:

1. **Survives context limits** — Claude Code compacts conversation history as it approaches the context window. File-based state means nothing is lost.
2. **Compounds across sessions** — the second run skips known issues and starts from the patterns learned in the first. Each run builds on the last.

## Limitations

- Optimized for TypeScript/Node.js (seed catalog and tracing patterns)
- Requires Claude Code (uses multi-agent subagent spawning)
- Each full investigation uses ~50-100K tokens across agents
- Static analysis only — doesn't execute code or reproduce bugs at runtime
- Confidence calibration is based on limited production runs
- Not a replacement for tests — finds the bugs you didn't know to test for
