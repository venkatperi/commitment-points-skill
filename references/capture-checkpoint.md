# Capture Checkpoint

Use when the change introduces a new business event type. Purpose: ensure the event gets written at event time, in the same transaction as the business action, so it can be queried, debugged, or billed against a month from now.

## The core rule

**Write at event time, atomic with the business action. Enrich later via update.**

Events are not recoverable upward. If the row is not written when the event happens, the data is gone. You will either backfill from a proxy (if you're lucky enough to have one) or apologize for numbers you cannot produce.

## The "month from now" test

For each new event, ask:

1. What will I need to know about this in a month when debugging a user complaint?
2. What will I need to know in a month when reporting to stakeholders or investors?
3. What will I need to know in a month when billing, auditing, or investigating fraud?

If any answer requires a field, that field is captured at event time. Not derived later. Not inferred from logs. Written in the same transaction as the business action.

## What counts as a business event

A business event is a state change the system or user cares about persisting. Common examples:

- user signed up
- user accepted terms / consent event
- user submitted X (question, form, message, file)
- payment initiated, succeeded, failed, refunded
- credit deducted, granted, transferred
- message sent, delivered, read
- session started, ended, expired
- content published, edited, deleted
- flag, report, moderation action
- feature access granted, revoked
- rating, vote, feedback submitted
- subscription started, renewed, canceled

Not business events: log lines, metrics samples, debug traces. Those go to observability systems. Business events go to durable storage.

## Event enumeration template

For each event, fill the artifact table:

| Field | Required value |
|---|---|
| Event name | Verb-past-tense: `question_submitted`, `payment_succeeded` |
| Table | Exact name of the table receiving the write |
| Atomic with business action? | yes / no. If no, this is a blocker. |
| Fields at event time | List specific columns |
| Fields enriched later | Columns updated via later UPDATE |
| Reconstruction fallback | Name the proxy, or `none` |

## Atomicity examples

**Good (atomic):**

```sql
BEGIN;
UPDATE users SET credits = credits - 1 WHERE id = $1;
INSERT INTO credit_transactions (user_id, amount, kind, created_at) VALUES (...);
INSERT INTO question_analytics (user_id, category, verdict, genuineness, created_at) VALUES (...);
COMMIT;
```

All three writes succeed together or fail together. The analytics row exists for every question the user was actually charged for. Reporting is accurate by construction.

**Bad (deferred):**

```python
deduct_credit(user_id)              # Writes users + credit_transactions
cast_chart(...)                     # No write
return_response(...)                # No write
# Later, in _finalize_session():
write_analytics(...)                # Only fires on explicit completion
```

If the session expires via TTL or the user closes the tab, the analytics row is never written. The event happened, the money moved, the record is lost. This is the AskPrasna capture failure in one block.

## When a business event has no atomic partner

Some events happen outside a database transaction: webhook arrives, async job completes, external API responds, cron fires. The rule still applies: at least one durable write tied to the event, made before any branching logic that could drop it.

Patterns:

- **Append-only event log.** Write to an `events` table at the moment the event is recognized, before any side effects. If the side effects fail, the event is still recorded with status.
- **Outbox pattern.** Write the event row in the same transaction as the state change. A separate process publishes to external systems. The DB write is the commitment; publishing is the delivery.
- **Idempotent webhook handler.** Write a row keyed on the source's idempotency ID (e.g. Stripe `payment_intent.id`) in the same transaction as the state change the webhook triggers. Repeat webhooks deduplicate naturally.
- **Async job start/complete pair.** Write `job_started` row when the job is kicked off, `job_completed` when it finishes (or `job_failed` with error). Lets you detect hung jobs.

## Common misses

- **Analytics on session finalize, not creation.** If sessions can expire without explicit finalize, you lose most events. Write on creation, enrich on finalize.
- **Payment tracking only on success.** The webhook for "payment failed" also needs a row, or you can't see churn, retry chains, or fraud patterns.
- **Feedback tables that don't reference the specific turn.** Thumbs up on "which interpretation" becomes unanswerable a month later. Capture enough foreign keys at event time.
- **Signups logged without attribution.** Source, campaign, referrer, invite code — all must be on the row at creation. Not reconstructable later.
- **LLM calls not logged.** Model, input tokens, output tokens, cost estimate, latency. Without these at call time, cost attribution is impossible.
- **Soft deletes without a timestamp.** Knowing something was deleted is useful. Knowing WHEN it was deleted is often what a support case depends on.
- **Config changes without a versioned record.** "Who turned off feature X and when" is a common incident-response question. If config changes are just updates to a row, the history is gone.

## Reconstruction fallbacks

If you already shipped without atomic capture and need to recover, these are backfills, not substitutes:

- **Credit movements proxy question submissions** — only if credits are always deducted on submit and there's a 1:1 relationship
- **Login events proxy sessions** — weak, because sessions and logins are not 1:1
- **Outbound webhooks proxy status changes** — only if the webhook is required and its delivery is durable
- **Application logs** — fine for debugging, unreliable for reporting (retention, sampling, parsing cost)

Write the real event row in every new code path. Do not ship new features with "we'll add analytics later."

## Artifact fields to fill

- **New events:** one per row
- **For each:** table + transaction + fields at event time
- **For each:** confirm atomic with business action
- **For each:** enrichment plan if fields update later
- **Any event without an atomic write:** blocker — resolve before implementation
