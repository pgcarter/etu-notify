# Look up before creating only when the previous outcome is unknown

Retries must not create duplicate certification records, and we can't assume the HR platform accepts an idempotency key (A5). A first attempt from a fresh pending completion creates the record directly. A completion whose previous attempt timed out, returned a 5xx, or was left in sending when its lease expired has an unknown outcome. Its next attempt first searches the employee's certification records for a match, and only creates one if none exists. A successful create saves the remote record ID. This keeps the normal cost to about one request per completion, which matters because each connection is limited to 100 requests/minute, while still covering the timeout-after-accept case.

## Considered Options

- **Look up before every create**: always safe, but it roughly doubles request usage during compliance season, when the rate limit is already the bottleneck.
- **Platform idempotency key or external reference**: preferred if the API supports it (A5); it replaces the lookup entirely.

## Consequences

- Sending is a stored status claimed in a short transaction, never held open across the HTTP call. An expired sending lease is how a crashed attempt is recognised as an unknown outcome.
- Outcome writes are conditional on the worker still holding the lease, but a remote record ID that comes back is always saved.
