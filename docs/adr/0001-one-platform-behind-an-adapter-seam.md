# One HR platform behind an adapter seam, not a generic gateway

We build completion sync for the one HR platform we already integrate with. The delivery machinery (ledger, scheduling, rate limiting, retries, token lifecycle, status) stays platform-agnostic, and everything platform-specific sits behind a thin adapter. A second platform then means writing one adapter, not a redesign. We rejected a generic multi-platform gateway because three engineers have six weeks to the first customer and ten to compliance season, and only one platform is actually needed.

## Considered Options

- **Generic gateway with a plugin model and canonical event schema**: speculative generality; it spends the timeline on platforms nobody has asked for yet.
- **Hard-coded for this platform, no seam**: fastest, but it ties retry and idempotency logic to one vendor's API and makes a second platform a rewrite.
