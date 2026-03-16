---
name: seeds-typescript-node
description: Pre-built investigation seed questions for TypeScript and Node.js codebases. Use when threadtrace needs seed questions for a TypeScript/Node.js project. Organized by damage tier from financial/data integrity (highest) to data transformation (subtle).
---

# Seed Questions: TypeScript / Node.js

Pre-built investigation seeds for TypeScript and Node.js codebases. These are starting points ranked by typical damage potential. Customize based on the specific domain.

## Tier 1: Financial / Data Integrity (Highest Damage)

### Database Write Failures
- "What happens when a database write fails in [critical operation]?"
- "What happens when a transaction partially commits?"
- "What happens when the database connection drops mid-operation?"
- "What happens when a unique constraint violation occurs in [write operation]?"

### Payment / Billing
- "What happens when the payment gateway returns a non-200 response?"
- "What happens when the payment succeeds but the local record fails to save?"
- "What happens when a webhook delivery fails?"
- "What happens when the idempotency key doesn't match?"

### Queue / Job Processing
- "What happens when a queue job fails after partial processing?"
- "What happens when the job retry compares a value that was regenerated?"
- "What happens when the worker crashes mid-job?"
- "What happens when the dead letter queue handler fails?"

## Tier 2: Authentication / Authorization (High Damage)

### Auth Flows
- "What happens when token refresh fails silently?"
- "What happens when the session store is unreachable?"
- "What happens when OAuth callback receives an error?"
- "What happens when JWT verification throws vs returns null?"

### Permission Checks
- "What happens when the permission check throws an exception?"
- "What happens when the user role lookup returns undefined?"
- "What happens when middleware throws before the auth check?"

## Tier 3: External Service Integration (Medium-High Damage)

### API Clients
- "What happens when the external API returns a 500?"
- "What happens when the API returns 200 with an error in the body?"
- "What happens when the API times out?"
- "What happens when the response doesn't match the expected schema?"

### Email / Notifications
- "What happens when the email service fails to send?"
- "What happens when the notification is marked as sent but the API returned an error?"
- "What happens when the template rendering fails?"

### File / Storage Operations
- "What happens when the file upload fails after the database record is created?"
- "What happens when S3 returns a 503?"
- "What happens when the presigned URL generation fails?"

## Tier 4: Async Patterns (Medium Damage, Hard to Detect)

### Promise Handling
- "Are there any async functions called without await?"
- "Are there any .then() chains where a step doesn't return?"
- "Are there any .catch() handlers that don't re-throw?"
- "Are there try/catch blocks wrapping non-awaited async calls?"

### Event Handling
- "What happens when an event listener throws?"
- "Are there any EventEmitter 'error' events without listeners?"
- "What happens when a stream errors mid-processing?"

### Callbacks
- "Are there any callbacks that aren't called on error paths?"
- "Are there any callback-based APIs mixed with promises incorrectly?"

## Tier 5: Data Transformation (Subtle Damage)

### Type Coercion
- "Are there any == comparisons with values that could be different types?"
- "Are there any .find() calls with {} bodies but no return statement?"
- "Are there any optional chaining (?.) chains that produce undefined used as truthy?"

### Validation
- "What happens when input validation throws instead of returning an error?"
- "What happens when a required field is undefined vs null vs empty string?"
- "Are there any parseInt/parseFloat calls without radix or NaN checks?"

## How to Use These Seeds

1. **With domain context**: After Icarus scans the codebase, pick seeds from the tier most relevant to the product's domain. A payment app → Tier 1 payment seeds. A SaaS platform → Tier 2 auth seeds.

2. **Without domain context**: Start with Tier 4 (async patterns) — these are universal in Node.js and almost always reveal something.

3. **After a finding**: Each confirmed bug suggests related seeds. A swallowed database error → check all other database operations. A missing await → check all other async calls in the same module.

4. **Custom seeds**: The best seeds come from domain understanding. "What happens when [the most valuable operation in this product] fails at [the most unreliable step]?" is always a strong starting point.
