<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/logo-dark.svg">
    <source media="(prefers-color-scheme: light)" srcset="assets/logo-light.svg">
    <img alt="threadtrace" src="assets/logo-light.svg" width="400">
  </picture>
</p>

<p align="center">
  <strong>Find the bugs that break production while every dashboard shows green.</strong>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT License"></a>
  <a href="https://github.com/onkardahale/threadtrace/stargazers"><img src="https://img.shields.io/github/stars/onkardahale/threadtrace?style=social" alt="GitHub Stars"></a>
  <a href="https://github.com/onkardahale/threadtrace"><img src="https://img.shields.io/badge/Claude%20Code-Plugin-blueviolet" alt="Claude Code Plugin"></a>
</p>

<p align="center">
  <a href="#what-it-finds">What It Finds</a> &middot;
  <a href="#install">Install</a> &middot;
  <a href="#how-it-works">How It Works</a> &middot;
  <a href="#limitations">Limitations</a>
</p>

threadtrace is a Claude Code plugin that traces error paths through your code — across files, into `node_modules/`, through library source code — to find correctness bugs that linters, type checkers, and tests all miss. Multi-agent investigation that reads actual library source instead of trusting documentation.

<p align="center">
  <img src="assets/demo.svg" alt="threadtrace finding a bug chain — dead retry logic masking a race condition" width="800">
</p>

## What It Finds

First run on a production TypeScript/Firebase/Hasura financial platform. ~50K LoC. All tests passing. Green dashboards. Zero errors in logs.

| Bug | What threadtrace found | Impact |
|-----|----------------------|--------|
| **Dead retry logic** | `.find()` with block body `{}` and no `return` — predicate always returns `undefined`, entire retry path is unreachable dead code | Every failed batch permanently loses records. No error, no log. |
| **Masked race condition** | 3 unawaited async calls create a race condition — currently hidden by the dead retry above. Fix one without the other and it gets worse. | State corruption, orphaned cloud resources, data loss |
| **Non-atomic user creation** | Insert user → set JWT claims → notify. No rollback. Auth triggers don't retry. Insert has no `on_conflict`. | Users locked out permanently. Manual DB fix required. |
| **Claims overwrite landmine** | Commented-out trigger uses `setCustomUserClaims` which OVERWRITES (not merges). Re-enabling it destroys auth state. | Dormant bug waiting for a future developer to activate. |

6 confirmed bugs across 3 seeds. Zero false positives. Each independently verified by a separate agent with no knowledge of how the bug was discovered.

## Why These Bugs Exist

Linters check syntax. Type checkers check types. Tests check the paths you thought to test. Security scanners check known vulnerability patterns.

None of them trace what actually happens when a database write fails at 3 AM — through your error handler, into the retry logic, across the library boundary, and back up to the caller that assumes success.

threadtrace does.

## Install

```bash
/plugin install threadtrace@onkardahale/threadtrace
```

## Run

```bash
# Full investigation — auto-scans, auto-seeds, auto-continues
/threadtrace:threadtrace

# With a specific seed
/threadtrace:threadtrace "what happens when the payment write fails?"

# Quick single-path trace
/threadtrace:threadtrace-trace "src/handlers/payment.ts error path"

# Quick verification of a claim
/threadtrace:threadtrace-verify "the retry in queue.ts never fires"
```

No config. No API keys. It scans your codebase, picks the highest-damage error path, and starts tracing.

## How It Works

Four agents with independent context. The agent that discovers a potential bug is never the agent that verifies it.

```
/threadtrace:threadtrace
│
├── Scout (haiku, fast)
│   Scans codebase → critical ops, boundaries, error hotspots
│
├── Thread Tracer (one per path, parallel)
│   Traces error path end-to-end INTO node_modules/
│   Reads library source code. Neutral: documents behavior, doesn't judge.
│
├── Pattern Architect (sonnet)
│   Compares against 3-5 sibling functions + git history
│   "4/5 re-throw, this one swallows. Sibling was fixed 2 weeks ago."
│
├── Hypothesis Formation (orchestrator)
│   Mechanism + downstream effect + blast radius + confidence score
│
└── Independent Judge (one per hypothesis, parallel)
    Receives ONLY: "investigate whether X handles Y correctly" + file paths
    Never sees discovery reasoning. Tries to disprove. Writes test case.
```

Context isolation between discovery and verification eliminates confirmation bias — structurally, not with prompt tricks.

## Persistent Memory

threadtrace gets smarter across sessions. Findings, learned patterns, and verified library behaviors persist to `.claude/threadtrace/`. The second run skips known issues and focuses on new threads.

## Limitations

- Currently optimized for TypeScript/Node.js codebases (seed catalog)
- Requires Claude Code (uses multi-agent subagent spawning)
- Each full investigation uses ~50-100K tokens across agents
- Not a replacement for tests — finds the bugs tests can't express

## License

MIT
