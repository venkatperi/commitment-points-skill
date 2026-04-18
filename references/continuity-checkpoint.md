# Continuity Checkpoint

Use when the change introduces a new piece of user-facing state. Purpose: decide where the state lives BEFORE writing the reader and writer, so the wrong answer doesn't propagate through the codebase.

## The core question

For every piece of new state, answer:

> Same user, new device tomorrow. What should they see?

Three valid answers, each with a different storage requirement:

| Answer | Storage | Examples |
|---|---|---|
| Their previous state | Server (DB, returned in user/session payload) | Preferences, consent, votes, history, drafts, unread counts, progress, settings |
| Fresh state every time | Server session, Redis, or in-memory | Temporary UI context, ephemeral flags, one-shot banners |
| Nothing (device-local by intent) | Browser local/session storage | Device theme override, "don't show again on THIS device", draft autosave for in-flight form, per-device accessibility tweaks |

If the answer is "their previous state," the storage is server-owned. **There is no MVP exception.** Retrofitting from browser storage to server is a full rewrite of reader and writer in every place the state is touched.

## State location matrix

Which layers survive which transitions:

| Location | Refresh | New tab | New device | Deploy | Clear cache |
|---|---|---|---|---|---|
| `useState` / component memory | No | No | No | No | No |
| `useRef` / module-level var | No | No | No | No | No |
| URL query params | Yes | Via URL share only | Via URL share only | Yes | Yes |
| `sessionStorage` | Yes | No | No | Yes | No |
| `localStorage` | Yes | Yes | **No** | Yes | **No** |
| IndexedDB | Yes | Yes | **No** | Yes | **No** |
| Cookies (non-httponly) | Yes | Yes | **No** | Yes | **No** |
| Server DB | Yes | Yes | **Yes** | Yes | Yes |
| Server cache (Redis) | Yes | Yes | Yes (within TTL) | Yes | Yes |

The "New device" column breaks most features. If the state needs to survive a device switch and it lives above "Server DB" in this table, you have a bug waiting for a user complaint.

## Common anti-patterns

- **Consent in localStorage.** Terms acceptance, cookie banners, onboarding completion. User signs in on a new device and sees the flow again. Must be server.
- **Preferences in `useState`.** Tier selection, sort order, filter choices. Feels "UI-ish" but the user expects persistence. Must be server if expected to persist.
- **Feedback, votes, reactions in component state.** Feels ephemeral because it's UI, but it's a business event AND the user expects it to persist. Must be server.
- **Draft content in component state only.** User starts typing, navigates away, loses the draft. Autosave to server, or at minimum IndexedDB with a clear "this device only" disclosure.
- **Session tier / mode / selection in client store but not session payload.** Works until refresh or second tab. Must come back from the server on every fetch of the session.
- **Shopping cart in localStorage.** Same problem. User adds items on phone, can't see them on laptop. Server-owned if it represents business intent.
- **Recently viewed / browsing history in localStorage when used for recommendations.** If the feature treats this as "user's" history, it must cross devices. If it's truly device-local (privacy-forward design), say so explicitly.

## Hybrid patterns

Some features legitimately want both:

- **Server source of truth + local cache for speed.** User preferences load from server on login, cached in IndexedDB for instant render on next load. Writes go to server first, then update cache on success. The cache is an optimization, not the source.
- **Server-owned with device-specific overrides.** Theme preference stored per-user on server, but a per-device override allowed for a specific session. Clear UX distinction: "use light mode on this device" vs "my theme preference."
- **Optimistic UI.** UI updates instantly from a mutation, server write happens in background, UI reconciles on response. The server is still the source; the client just doesn't wait to render.

If you're building one of these, say so in the artifact. "Server source of truth with local cache" is different from "localStorage only."

## Decision rules

- Default to server for anything a user would describe as "their" something (their preference, their rating, their draft, their history).
- Use browser storage only for truly device-local intent (per-device theme, "don't show again on THIS computer", in-flight form drafts).
- When in doubt, server. The cost of server storage is one column and one line in the user/session payload. The cost of wrong storage is a full rewrite.
- If the storage choice is disputed, ask the user the core question verbatim: *"Same user, new device tomorrow, what should they see?"* Their answer decides.

## What the server-owned plan needs to include

- Table name and column (or existing table + new column)
- Migration file
- Write endpoint (or inclusion in existing mutation)
- Read path (usually: returned as part of `/auth/me` or `/sessions/current`)
- Frontend component reads from returned payload, never from local storage
- Removal of any interim client-side persistence

If the state is cross-session but ephemeral (live session context like "currently viewing interpretation N"), Redis + session payload is appropriate. Still server-owned, just in cache not DB.

## Artifact fields to fill

- **New state items:** one row per distinct piece of state
- **For each:** "same user, new device" answer
- **For each:** chosen storage location with justification
- **For each:** migration / endpoint / payload changes needed
- **Device-local items:** explicit reason for device-local treatment
