---
name: threadtrace-verify
description: Quick verification — independently verify a claim about code behavior
user_invocable: true
memory: project
---

# threadtrace-verify — Quick Independent Verification

A lightweight version of threadtrace that independently verifies a single claim about code behavior. Use this when someone (human or AI) claims code behaves a certain way and you want independent confirmation.

## Usage

The user provides a claim to verify:
- `"the retry in queue.ts never fires"`
- `"processPayment swallows database errors"`
- `"the callback in createUser is never called on error"`

## Process

### Step 1: Parse the Claim

Extract from the user's input:
- The hypothesis: what behavior is being claimed?
- The location: what files/functions are involved?

If the location isn't specified, use Grep and Glob to find the relevant code.

### Step 2: Spawn Minos

Spawn **Minos** (the independent judge subagent) with a neutrally framed hypothesis:

```
Investigate the following hypothesis about this codebase:

HYPOTHESIS: Investigate whether [function/component] handles [scenario] correctly.

FILES TO INVESTIGATE:
[list of file paths]

Read the code yourself. Trace the error path independently. Try to disprove the hypothesis — look for upstream handling, retry mechanisms, intentional design. If you confirm it, write a concrete test case. If you disprove it, explain why with evidence.
```

**CRITICAL**: Frame neutrally. Do NOT say "verify this bug" or "confirm this problem." Say "investigate whether X handles Y correctly."

### Step 3: Present the Verdict

Present Minos's verdict directly to the user:
- **CONFIRMED**: The claim is correct. Here's the evidence and a test case.
- **DISPROVED**: The claim is incorrect. Here's why.
- **PARTIALLY CONFIRMED**: The claim is partially correct. Here's what's right and wrong.
- **INCONCLUSIVE**: Cannot determine. Here's what's missing.

### Step 4: Suggest Next Steps

Based on the verdict:
- If CONFIRMED: Suggest running full `/threadtrace:threadtrace` to assess blast radius and find related issues
- If DISPROVED: Explain what actually happens and why it's correct
- If PARTIALLY CONFIRMED or INCONCLUSIVE: Suggest specific follow-up investigations

## Notes

- This is an independent verification tool — Minos works with fresh context
- The claim is deliberately framed neutrally to avoid confirmation bias
- Minos will try to disprove the claim before confirming it
- If the claim is confirmed, Minos provides a concrete test case
