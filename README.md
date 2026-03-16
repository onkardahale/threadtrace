<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/logo-dark.png">
    <source media="(prefers-color-scheme: light)" srcset="assets/logo-light.png">
    <img alt="threadtrace" src="assets/logo-light.png" width="400">
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
  <a href="#proof">Proof</a> &middot;
  <a href="#how-it-works">How It Works</a> &middot;
  <a href="#install">Install</a> &middot;
  <a href="#limitations">Limitations</a>
</p>

<p align="center">
  <img src="assets/demo.gif" alt="threadtrace finding a bug chain — dead retry logic masking a race condition" width="800">
</p>

## Proof

First run on a production TypeScript/Firebase/Hasura financial platform. ~50K LoC. All tests passing. Green dashboards. Zero errors in logs.

| Bug | What threadtrace found | Impact |
|-----|----------------------|--------|
| **Dead retry logic** | `.find()` with block body `{}` and no `return` — predicate always returns `undefined`, entire retry path is unreachable dead code | Every failed batch permanently loses records. No error, no log. |
| **Masked race condition** | 3 unawaited async calls hidden by the dead retry above. Fix one without the other and it gets worse. | State corruption, orphaned cloud resources |
| **Non-atomic user creation** | Insert user → set JWT claims → notify. No rollback. Firebase v1 auth triggers don't retry — verified by reading the SDK source, not the docs. | Users locked out permanently. Manual DB fix required. |
| **Claims overwrite landmine** | `setCustomUserClaims` OVERWRITES all claims (doesn't merge) — discovered by reading the Firebase SDK source in `node_modules/`, not the documentation. | Dormant bug waiting for a future developer to activate. |

6 confirmed bugs across 3 seeds. All findings independently verified by a separate agent with no knowledge of how the bugs were discovered — no rejected hypotheses in this dataset.

## Why Nothing Else Catches These

Linters check syntax. Type checkers check types. Tests check the paths you thought to test.

None of them trace what actually happens when a database write fails at 3 AM — through your error handler, into the retry logic, across the library boundary, and back up to the caller that assumes success.

And none of them read library source code. Documentation says `setCustomUserClaims` "sets claims." The actual implementation **overwrites all existing claims**. Documentation says Firebase auth triggers "handle errors." The actual v1 SDK **does not retry on failure**. These aren't edge cases — they're the normal behavior of widely-used libraries that happens to differ from what the docs say.

threadtrace reads the actual code in `node_modules/`. That's how it finds bugs that exist in the gap between what you think a library does and what it actually does.

## How It Works

Most AI tools find bugs by convincing themselves they're right. threadtrace forces a second agent to try to prove them wrong.

```
/threadtrace:threadtrace
│
├── Scout (haiku, fast)
│   Scans codebase → ranks operations by "if this fails silently, how bad?"
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

The judge is a separate agent instance with no shared memory. It doesn't see how the hypothesis was formed, what the confidence score is, or what evidence was collected. Its default stance is that the code is correct. This isn't a prompt trick ("be skeptical") — it's architectural. The judge literally cannot access the discovery context because it was never provided.

[Full methodology →](METHODOLOGY.md)

## Install

```bash
git clone https://github.com/onkardahale/threadtrace.git ~/.claude/plugins/threadtrace
```

Then start Claude Code with the plugin:

```bash
claude --plugin-dir ~/.claude/plugins/threadtrace
```

## Run

```bash
# Full investigation — auto-scans, auto-seeds, auto-continues
/threadtrace:threadtrace

# Target a specific concern
/threadtrace:threadtrace "what happens when the payment write fails?"

# Quick single-path trace (no verification)
/threadtrace:threadtrace-trace "src/handlers/payment.ts error path"

# Verify a specific claim
/threadtrace:threadtrace-verify "the retry in queue.ts never fires"
```

Seed selection is automatic — the scout ranks operations by damage potential and picks the highest-risk path. You can also provide your own seed if you already suspect something.

## Persistent Memory

Findings, learned patterns, and verified library behaviors persist to `.claude/threadtrace/`. The second run skips known issues, avoids known false positive patterns, and starts from what was learned in the first. Each run compounds on the last.

## Limitations

- Currently optimized for TypeScript/Node.js codebases (seed catalog)
- Requires Claude Code (uses multi-agent subagent spawning)
- Each full investigation uses ~50-100K tokens across agents
- Static analysis only — traces code paths, doesn't execute them
- Confidence calibration is based on limited production runs
- Not a replacement for tests — finds the bugs you didn't know to test for

## License

MIT
