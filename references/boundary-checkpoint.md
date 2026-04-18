# Boundary Checkpoint

Use when the change introduces a new service, Lambda, API endpoint, external call, or LLM invocation. Purpose: surface platform-imposed limits that will break the chosen approach in production.

## The walkthrough

Trace the request path in production. For EACH hop the request crosses, name:

1. **Hop name** (client, CDN, API Gateway, Lambda, downstream service, LLM, DB)
2. **Timeout** imposed by that hop
3. **Typical latency** (p50) of what happens at that hop
4. **Worst-case latency** (p99 or hard maximum)
5. **What kills the request** at that hop when it exceeds the timeout (504, connection reset, silent drop)

The walkthrough output goes into the artifact's Boundary table.

## Known ceilings

Reference this catalog before answering "what's the timeout." Do not guess.

| Boundary | Limit | Failure mode |
|---|---|---|
| API Gateway (REST/HTTP) | **29 seconds** | 504 returned to client |
| API Gateway WebSocket | 2 hours idle, 10 min per message | Connection closed |
| Lambda function | 15 minutes | Invocation aborted |
| Lambda response payload (sync) | 6 MB | Truncated or error |
| Lambda response payload (async) | 256 KB | Error |
| Lambda concurrent executions | 1000 default per account | Throttled |
| CloudFront origin response | 60 seconds (configurable to 180) | 504 |
| ALB idle timeout | 60 seconds default | Connection dropped |
| Browser `fetch` (no AbortSignal) | None | Hangs indefinitely |
| SQS message visibility | 12 hours max | Redelivery |
| Step Functions standard | 1 year | N/A |
| Step Functions express | 5 minutes | Aborted |
| DynamoDB item size | 400 KB | Write rejected |
| DynamoDB BatchWrite | 25 items | Partial writes |
| RDS Postgres connection | pool dependent | Connection refused |
| Bedrock streaming | 15 minutes | Stream closed |
| Bedrock non-streaming | 60-300s typical | Request fails |
| Anthropic API streaming | 10 min idle | Stream closed |
| OpenAI API streaming | 10 min idle | Stream closed |

## LLM latency reference

LLM calls are the most common boundary violation. Ballpark numbers:

- **Claude Sonnet** (non-streaming, ~2K output tokens): 20-60s typical, 90s+ worst case
- **Claude Opus** (non-streaming, ~2K output tokens): 30-90s typical, 120s+ worst case
- **Claude Haiku** (non-streaming, ~500 output tokens): 2-8s typical
- **GPT-4** class (non-streaming, ~2K output): 15-45s typical
- **First-token latency (streaming)**: 500ms - 2s
- **Long context (100K+ input tokens)**: add 10-30s to any call
- **Structured output / JSON mode**: add 10-30% to non-streaming latency
- **Tool use roundtrips**: each tool call is a separate model invocation; multiply accordingly

Rule of thumb: any non-streaming LLM call behind API Gateway is at risk. Design async by default.

## The fit check

After the walkthrough:

- Find the slowest call on the path.
- Find the tightest ceiling above it.
- If `slowest_worst_case > tightest_ceiling`: the path WILL fail in production.
- If the worst case is unknown: the path MIGHT fail. Treat as fail until measured.
- If fit is within 2x of the ceiling: treat as fragile. Add safety margin.

## Decision rules

**Fit passes with margin (>2x headroom):** Proceed. Document the ceiling in code comments near the slow call so the next person sees it.

**Fit is tight (1-2x headroom):** Add a safety margin. Options:
- Set explicit client-side timeout at 80% of the ceiling
- Add retry with backoff inside the tight window
- Move the slow call async if worst-case latency grows with input size

**Fit fails:** The architecture must change before implementation. Common moves:

- **Make the long call async.** Return immediately with a job ID, poll or webhook on completion. This is the default for LLM calls behind HTTP APIs.
- **Split the work.** Break one long call into several shorter ones with intermediate state.
- **Move to a service with a higher ceiling.** API Gateway REST → WebSocket or Step Functions for >29s work.
- **Stream the response.** Works if the client can render progressively and the streaming path fits within ceilings (Lambda URLs or ALB, not API Gateway REST).

## Common misses

- **Synchronous LLM calls behind API Gateway.** Nearly every LLM-backed web API hits the 29-second wall. Async from day one.
- **Forgotten client timeouts.** Browser `fetch` has no default timeout; a frontend waiting on an already-dead Lambda looks like a hung page. Always pass an `AbortSignal` with a timeout below the server ceiling.
- **Lambda response size for LLM outputs.** 6 MB sync cap is reachable with verbose responses, embedded images, or chart data. Stream or paginate.
- **Cold starts on the slowest path.** VPC-bound Lambdas with heavy native deps (psycopg2, pyswisseph, cryptography) can take 5-10s to initialize. Factor into worst case.
- **Ignoring retries in latency budget.** If the client retries on failure, worst-case is `attempts × single_call_worst_case`. A 3x retry on a 30s call is a 90s total wait.
- **Downstream service timeouts below your own.** If you call an API with a 5s timeout and allocate 10s for the call, your 10s budget is meaningless.

## Artifact fields to fill

- **Request path:** ordered list of hops with timeouts, p50, p99, failure mode
- **Slowest call:** name and worst-case latency
- **Tightest ceiling:** hop name and limit
- **Fit verdict:** pass / fail / unknown-measurement-needed
- **Resolution (if fail or unknown):** specific plan, doc, or measurement
