# Delivery slices

Each slice goes end to end through the real system, adds to the slice before it, and finishes with an automated test. What's real in one slice stays real in every later slice. Assumption IDs (A1–A13) refer to `DESIGN-QUESTIONS.md`.

**Milestones:** first customer live after slice 8 (week 6). Slice 9 is done before compliance season (week 10).

**Day one, alongside slice 1:** send the A1–A3 questions to the legacy team and the A12/A13 questions to the HR vendor. If A12/A13 are still unanswered at the end of week 2, that becomes the top risk.

## The port

The core never sees HTTP. It talks to the HR platform through one port that returns delivery outcomes:

```ts
interface HrPlatform {
  deliver(connection, completion, opts: { previousOutcomeUnknown: boolean }):
    Promise<
      | { kind: 'delivered'; remoteRecordId: string }
      | { kind: 'retry_later'; after?: Date; outcomeUnknown: boolean }
      | { kind: 'needs_action'; reason: NeedsActionReason }
    >
}
```

The adapter translates HTTP responses into these outcomes, and does look-up-before-create internally when `previousOutcomeUnknown` is true (ADR 0004).

Slices 1–5 use an **in-memory fake** of this port. A **shared contract test suite** runs against the fake from slice 1, and against the real HTTP adapter from slice 6, so the fake can't drift from the real adapter unnoticed.

Tokens sit behind the adapter's side of the seam. The HTTP adapter gets its access tokens from a `TokenProvider`; the core never sees tokens, and the in-memory fake doesn't need them:

```ts
interface TokenProvider {
  accessToken(connection): Promise<string>   // single-flight refresh inside
  invalidate(connection): void               // called by the adapter on a 401
}
```

---

## 1. Happy path from an event

**Scope:**
- An event format we define, sent to our own ingest point
- Syncable check: mapped course, sync on, learner has an employee identifier
- Completion ledger row in Pending
- Worker claims it as Sending in a short transaction
- The port's `deliver` call
- The row becomes Delivered, with the remote record ID saved

**Real data:** the integration service's own tables, filled with test data: one connection, one learner, one course mapping. Only the HR platform is faked.

**Assume everything works:** no failures, no tokens, no rate limit, one connection. The fake always returns `delivered`.

**Contract suite:** created now, with happy-path cases only.

**Done when:**
- *Given* a seeded connection, learner and course mapping, *when* a completion event arrives, *then* the fake holds exactly one certification record for that employee with the matching certification type and date, and the ledger shows Delivered with the fake's remote record ID.
- Sending the same event a second time still leaves exactly one record.

**Blocked on:** nothing.

## 2. Failure handling (delivery and ingestion)

**Delivery:**
- `retry_later`: the completion goes back to Pending with exponential backoff and jitter, honouring `after`.
- `needs_action`: the completion moves to Needs action with its reason.
- After about 24 hours of retries, it moves to Needs action with reason `retries_exhausted`.
- An expired Sending lease means the outcome is unknown, so the next attempt passes `previousOutcomeUnknown`.
- Outcome writes are fenced on `lease_owner`, but a returned remote record ID is always saved.
- The send-time eligibility check moves a completion to Skipped with a reason.
- Re-queueing Needs action completions, one at a time or in bulk.

**Ingestion:**
- A completion that isn't syncable is ignored, not recorded.
- A malformed event is logged and counted.
- Until slice 4, a lost event is genuinely lost. That's acceptable because no customer is live yet.

**What the fake gains:**
- A script of outcomes per call (e.g. three `retry_later`, then `delivered`).
- An "accept but report unknown" mode: it stores the record and returns `retry_later` with `outcomeUnknown: true`.
- The test harness can stop a worker partway through a delivery.

**Done when:**
- A completion scripted to fail and then succeed ends Delivered.
- In accept-but-report-unknown mode, the completion ends Delivered with exactly one record.
- A worker stopped mid-delivery leaves an expired lease, the completion is reclaimed, and it ends with exactly one record.
- Each needs-action reason ends in Needs action with that reason.

**What this proves and doesn't:** it proves the **core** handles every outcome correctly. It doesn't prove the HTTP adapter maps real responses to the right outcomes; that's covered by the contract suite in slice 6.

**Blocked on:** nothing.

## 3. Send rate

**Scope:**
- A token bucket per connection, refilling at about 80 a minute, below the platform's 100 to leave room for lookups and token refreshes. A worker takes a token before claiming rows; with none available, the connection is skipped until its next refill.
- A `retry_later` that came from a 429 pauses the **whole connection** (`paused_until = now + Retry-After`), not just that completion.

**What the fake gains:** it counts calls per connection per minute, and returns `retry_later` with an `after` time once there are more than 100.

**Done when:**
- A burst of completions for one connection never pushes the fake above 100 calls a minute, and never causes a 429.
- When the fake does return a 429, no completion for that connection is sent until `paused_until`, and then delivery continues.

**Blocked on:** nothing.

## 4. Sweep and the real legacy event

**Scope:**
- A reconciliation sweep per customer every 15 minutes. It asks the legacy internal API for completions since `watermark − overlap`, with a 1-hour overlap and the interval configurable.
- Each completion found is recorded through the same step slice 1 uses.
- The real legacy completion event, translated into the format slice 1 already accepts.
- A metric counting completions the sweep found that the event missed.

**Done when:**
- A completion whose event was dropped is found by the sweep and ends Delivered.
- A completion saved late, with a time before the watermark, is still found because of the overlap.
- Overlapping sweeps and duplicate events still produce exactly one record.

**Blocked on:** A1 (can the legacy API list a customer's completions since T?), A2 (a stable completion ID), A3 (how the event is delivered and what it contains).

## 5. Tokens

**Scope:**
- A `TokenProvider` module, used by the HTTP adapter in slice 6, not by the core.
- Proactive refresh when under 10 minutes remain.
- Single-flight refresh, locked on the connection row, re-checking expiry inside the lock.
- The new token pair saved (fenced write) before it's used. If tokens live in Secrets Manager, the secret's `VersionId` is recorded on the connection row and checked before writing (A10).
- `invalidate(connection)`, so the adapter can force a refresh after a 401.
- A scheduled keepalive job that refreshes connections idle more than 30 days, so the 60-day expiry never hits.
- `invalid_grant` marks the connection `needs_reauth`; the adapter then returns `needs_action: auth_revoked`, and the completions wait in the ledger.

**What we build to test it:** a fake OAuth token endpoint that rotates the refresh token on every use and rejects a reused refresh token.

**Done when:**
- Race test: N concurrent callers cause exactly one refresh, and every caller gets the new token.
- Reusing an already-used refresh token fails as `invalid_grant` and marks the connection `needs_reauth`, without a retry loop.
- Keepalive refreshes a connection that has been idle for 31 days.

**Blocked on:** nothing hard. Proceed with sensible defaults, and adjust when A6 (do refreshes count against the limit?), A7 (what happens when a used refresh token is reused?) and A10 (where tokens are stored) are answered.

## 6. Real HTTP adapter

**Scope:**
- The HTTP adapter for the vendor API. It gets access tokens from the `TokenProvider` (slice 5) and:
  - maps HTTP responses to delivery outcomes (429 → `retry_later` with `after`; timeout or 5xx → `retry_later` with `outcomeUnknown`; unknown employee → `needs_action`; 401 → `invalidate` and retry once)
  - does look-up-before-create when `previousOutcomeUnknown` is true, or uses the vendor's idempotency key if it has one (A5)
- The contract suite runs against the HTTP adapter, using the vendor sandbox or recorded responses.

**Done when:** the contract suite passes against both the in-memory fake and the HTTP adapter.

**Blocked on:** A12 (sandbox), A13 (API docs), A5 (idempotency support).

## 7. Backfill

**Scope:**
- Backfill is the sweep with the watermark set to the beginning. It runs when a customer turns sync on, and when a course mapping is added.
- A backfill preview reports how many completions will be sent and how many can't be (no employee identifier, mappings to check), without sending anything.
- Sending starts only after a person approves.
- Backfill completions have backfill priority, so they only use the connection's spare send budget after live completions.
- A feature flag per customer.

**Done when:**
- A preview reports the correct counts and sends nothing.
- After approval, every historical syncable completion ends Delivered.
- Live completions arriving during a large backfill are delivered first.

**Blocked on:** A9 (confirm that a mapping added later should trigger a backfill).

## 8. Customer success view and alerts

**Scope:**
- Search by customer and learner. Each completion shows its sync status, reason, attempt count, last error, next attempt time and remote record ID.
- Every needs-action reason has plain-English text and a "what to do" line.
- A re-queue action on Needs action completions.
- Completions with reason `auth_revoked` are re-queued automatically when the customer reconnects.
- Alerts:
  - oldest Pending age per connection
  - Needs action count, by reason
  - token health (days until each refresh token expires)
  - sweep catches, and how long since each customer's last successful sweep
- Routing: systemic problems page engineering; problems for one customer go to customer success.

**Done when:** a customer success person can answer "did this completion sync, and if not, why?" for a seeded scenario covering every sync status, without help.

**Blocked on:** A11 (is there existing internal admin tooling?).

**→ First customer live (week 6).**

## 9. Several connections

**Scope:**
- Workers take turns across connections that have due work and aren't paused, so one large customer can't starve the others.
- A compliance-season load test: 8,000 completions an hour, concentrated in 3 customers, against the fake's rate limits.
- A chaos simulation: thousands of completions, with random worker kills, injected failures and dropped events.

**Done when:**
- Load test: small customers' oldest Pending age stays low while large customers have backlogs. Each large customer's drain time is measured and agreed with customer success as what "promptly" means at peak.
- Chaos simulation: every completion ends in exactly one visible sync status, and the fake holds at most one certification record per completion.

**Blocked on:** nothing.

**→ All large customers' backfills finished before compliance season (week 10).**
