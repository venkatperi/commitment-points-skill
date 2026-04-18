# Commitment check: `<feature-name>`

**Date:** `<YYYY-MM-DD>`
**Triggers fired:** `<boundary | continuity | capture>` (list all that apply)
**Feature summary:** `<one-sentence description of what's being built>`

---

## Boundary

*Include this section only if the boundary trigger fired. Delete otherwise.*

**Request path (production):**

| Hop | Timeout | p50 | p99 / worst case | Failure mode |
|---|---|---|---|---|
| | | | | |

**Slowest call:** `<hop name, worst-case latency>`
**Tightest ceiling:** `<hop name, limit>`
**Fit:** `pass` / `fail` / `unknown-measurement-needed`

**If fail or unknown, resolution:**

- `<specific action, measurement, doc to read, or architectural change>`

---

## Continuity

*Include this section only if the continuity trigger fired. Delete otherwise.*

**New state items:**

| Item | Same user, new device, what should they see? | Storage | Changes needed |
|---|---|---|---|
| | | | |

**Storage key:**
- Server DB: persists across devices, refresh, deploys
- Server cache (Redis): persists across devices within TTL
- localStorage / sessionStorage: device-local only
- Component state (useState, useRef): lost on refresh

**Device-local items (explicit reasons):**

- `<item>`: `<reason it should be device-local>`

---

## Capture

*Include this section only if the capture trigger fired. Delete otherwise.*

**New business events:**

| Event | Table | Atomic with business action? | Fields at event time | Enriched later |
|---|---|---|---|---|
| | | `yes` / `no` | | |

**Any `no` in the "atomic" column is a blocker.** Resolve before writing implementation code.

---

## Decision

- [ ] **Proceed** — all fits pass, all state locations decided, all events atomic
- [ ] **Revise** — `<list specific items to fix>`
- [ ] **Blocked on** — `<measurement, question, or user input needed>`

**Next step:** `<what happens after this artifact is approved>`
