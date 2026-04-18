# AskPrasna — Worked Examples

Three real commitment-point misses from a recent build session. Each shows the feature as initially stated, what the artifact would have produced, the decision it would have forced, and what actually happened when the check was skipped.

Use these when the current feature resembles one of the patterns. Reference them to help the user recognize they're looking at the same mistake.

---

## Example 1: The 29-second wall

**Feature as stated:** "User submits a question, we cast a chart and return an interpretation."

**Triggers fired:** boundary (new API endpoint + new LLM call), capture (question event)

### What the boundary artifact would have caught

| Hop | Timeout | p50 | p99 / worst case | Failure mode |
|---|---|---|---|---|
| Browser fetch | none set | — | — | Hangs |
| CloudFront | 60s origin | ~100ms | ~100ms | 504 |
| API Gateway | **29s** | — | — | 504 to client |
| Lambda (API handler) | 30s | ~500ms | ~3s | Invocation error |
| Lambda (Interpretation) | 60s | ~30s | **30-60s** | Invocation error |
| Bedrock Claude Sonnet | 300s | ~30s | 60s+ | Request fails |

**Slowest call:** Bedrock Sonnet invocation, 60s worst case
**Tightest ceiling:** API Gateway, 29 seconds
**Fit:** **fail** — the LLM call exceeds API Gateway's ceiling by 2x at worst case

### Decision the artifact would have forced

The interpretation path cannot be synchronous behind API Gateway. Two options:

1. Fire the Interpretation Lambda async (`InvocationType='Event'`), return session ID immediately, frontend polls `/sessions/current`
2. Use WebSocket API or Step Functions for the long path

Async polling is simpler for this use case. Implementation shape: `POST /questions` creates the session in `interpreting` state, fires async Lambda, returns in ~3 seconds. Frontend polls every 2 seconds, session flips to `active` when the Interpretation Lambda writes back to Redis.

### What actually happened without the check

Built synchronously. Worked in dev. Shipped to staging. 504 errors. Rewrote the API route, the Interpretation Lambda, the Redis client, the `Session` type, added a polling hook, rewrote the progress UI, changed the Home redirect logic. Seven files.

Then follow-up questions hit the same wall and the refactor repeated.

The artifact would have taken five minutes to produce. The rewrite took hours across two sessions.

---

## Example 2: State that vanishes on device switch

**Feature as stated:** "Add a terms-and-conditions modal on first login."

**Triggers fired:** continuity (new user state), capture (consent event)

### What the continuity artifact would have caught

| Item | Same user, new device, what should they see? | Storage | Changes needed |
|---|---|---|---|
| Terms accepted flag | They already accepted on laptop — don't show again on phone | **Server (users.terms_accepted_at)** | Migration + `/auth/terms-accept` endpoint + include in `/auth/me` response + modal reads from user payload |

### Decision the artifact would have forced

Server-owned from day one. `localStorage` is not a valid storage location for consent.

### What actually happened without the check

Shipped with `localStorage`. User signed in on phone. Saw the modal again. User complaint arrived within a day. Fix required: migration + model function + API endpoint + CDK wiring + modal rewrite. About 45 minutes.

The pattern then repeated twice in the same session: feedback votes (useState → Postgres, another 45 minutes), and session tier preference (store-only → session payload, another 20 minutes).

### Meta-lesson

After the first continuity miss, the practice should have been: never ship user state without running the continuity check. One artifact per new state item. The three-peat happened because the checkpoint wasn't institutionalized after the first failure.

---

## Example 3: Analytics you can't recover

**Feature as stated:** "Track question events for the admin dashboard."

**Triggers fired:** capture

### What the capture artifact would have caught

| Event | Table | Atomic with business action? | Fields at event time | Enriched later |
|---|---|---|---|---|
| `question_submitted` | `question_analytics` | Originally planned: **no** (finalize only) | user_id, category, verdict, genuineness, created_at | followup_count, duration |

**Blocker:** atomic = no. Sessions expire via 30-minute Redis TTL more often than they explicitly finalize. The event would be lost for the majority of sessions.

### Decision the artifact would have forced

Write the analytics row at question creation, in the same transaction as the credit deduction. Update it at finalize if and when finalize happens.

Implementation shape:

```sql
BEGIN;
UPDATE users SET credits = credits - 1 WHERE id = $1;
INSERT INTO credit_transactions (...) VALUES (...);
INSERT INTO question_analytics (user_id, category, verdict, genuineness, created_at)
  VALUES (...);
COMMIT;

-- Later, on session finalize:
UPDATE question_analytics
  SET followup_count = $1, duration_seconds = $2
  WHERE session_id = $3;
```

### What actually happened without the check

Shipped with `write_analytics()` inside `_finalize_session()`. Admin dashboard showed near-zero sessions for three days. Fix: recompute session counts from `credit_transactions` as a proxy.

The business event (question submitted) could be proxied from credit movements, because credits were always deducted atomically. But the derived metadata (category, verdict) was lost entirely — no proxy could reconstruct what the LLM decided at submission time.

### Meta-lesson

Backfill works for the business event itself when there's a reliable atomic partner (credit deduction). Backfill does NOT work for derived data that's only computed at event time. If it isn't captured when the event happens, it's gone.

---

## How to use these examples

- When the current feature involves a synchronous LLM call, reference Example 1. Ask: *"Does this look like the 29-second wall?"*
- When the current feature involves per-user state and the user said anything like "it's just UI state" or "we can put it in localStorage for now," reference Example 2.
- When the current feature involves any event that should be reportable a month from now, reference Example 3.

Paste the artifact tables directly into the check for the current feature. Change the fields. The structure is the point.
