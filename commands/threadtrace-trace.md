---
name: threadtrace-trace
description: Quick single-path trace — trace one error path through code
user_invocable: true
memory: project
---

# threadtrace-trace — Quick Error Path Trace

A lightweight version of threadtrace that traces a single error path without the full investigation cycle. Use this when you know exactly what path you want to trace.

## Usage

The user provides a target to trace:
- A file path and error scenario: `"src/handlers/payment.ts error path"`
- A function name: `"processPayment failure handling"`
- A question: `"what happens when the DB write in createUser fails?"`

## Process

### Step 1: Locate the Target

If the user gave a file path, read it directly. If they gave a function name or question, use Grep and Glob to find the relevant code.

### Step 2: Spawn Theseus

Spawn **Theseus** (the thread tracer subagent) with the target:

```
Trace the following error path end-to-end through the code:

SEED: [user's question/target]

Start at: [file paths you identified]

Trace what happens when this operation fails. Follow the error through every handler, across files, and INTO dependency source code in node_modules/. Document exactly what the code does at each step. Do NOT classify anything as a bug — just document the behavior neutrally.

Return an ordered trace with file:line references and library source quotes.
```

### Step 3: Present the Trace

Present Theseus's trace directly to the user. Add a brief summary at the top highlighting:
- How many files the trace crosses
- Whether library source was consulted
- Any open questions Theseus flagged

### Step 4: Suggest Next Steps

Based on the trace, suggest:
- Run full `/threadtrace:threadtrace` with seeds derived from this trace
- Run `/threadtrace:threadtrace-verify` on any suspicious behavior found
- Specific follow-up traces for paths Theseus flagged as open questions

## Notes

- This is a quick tool — no pattern comparison, no hypothesis formation, no verification
- Theseus's output is neutral (documents behavior, doesn't judge)
- If the trace reveals something worth investigating further, recommend the full `/threadtrace:threadtrace` flow
