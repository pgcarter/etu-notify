# Design questions and assumptions

Things the design depends on that the brief doesn't settle. Each one says what we assume, why it matters, and what changes if the assumption is wrong. The **Ask** items are questions to bring to the session.

## Legacy learning platform

### A1. The internal API can list a customer's completions since a timestamp
- **Why it matters:** the reconciliation sweep and backfill both depend on it (ADR 0002).
- **Ask:** Can it filter by customer and an updated-since time, and does it paginate? How heavily can we call it (it's a legacy ColdFusion service)?
- **If wrong:** ask for that endpoint as the one change to the legacy platform, or sweep per learner (slow, and heavy on the legacy service), or fall back to a nightly export.

### A2. A completion is identified by (legacy record ID, completed-at)
- **Why it matters:** this is the idempotency key in our ledger. The brief doesn't say the legacy platform has a completion ID at all. If a re-completion (e.g. an annual refresher) updates the existing record in place, a key on the record ID alone would treat it as a duplicate and silently drop it.
- **Decision:** key the ledger on (legacy record ID, completed-at). This catches re-completions whether the legacy platform writes a new record per attempt or overwrites one record per learner and course.
- **Known limitation:** an admin correcting a completion date produces a new key, so a second certification record is sent. This is rare and visible to customer success, because both ledger rows share the record ID. If it proves a problem, treat a new date close to an existing delivered row for the same record as a correction (Needs action `date_changed`) rather than a re-completion.
- **Ask:** Is a re-completion a new record, or an update to the existing one? Are completion dates ever edited after the fact? Is there a record ID at all, or only (learner, course, timestamp)?
- **If wrong:** with no record ID, derive one from (customer, learner, course). The key is then (derived ID, completed-at), and nothing else changes.

### A3. We can receive the completion event in a form we control
- **Why it matters:** decides the ingestion path.
- **Ask:** Where does the event go today (SNS, SQS, HTTP webhook, nothing yet)? What does it contain?
- **If wrong:** if it can only call an HTTP endpoint, we expose a thin ingest endpoint that writes straight to the ledger.

### A4. Revocations are out of scope for v1
- **Ask:** Do completions ever get reversed, and does the HR platform need to know?
- **If wrong:** add a revocation record to the ledger and a delete or void call to the adapter.

## HR platform API

### A5. Idempotency on the HR side
- **Ask:** Does the create-certification-record endpoint accept an idempotency key or an external reference we choose? Can we search an employee's certification records?
- **If wrong:** look up existing records before writing (this costs rate limit) and save the remote record ID once a write succeeds.

### A6. Does refreshing a token count against the 100 requests/minute limit?
- **Why it matters:** this sets the real throughput per connection.

### A7. How does the platform handle reuse of a rotated refresh token?
- **Ask:** If an old refresh token is used, does it just fail, or is the whole token chain revoked?
- **Why it matters:** decides how badly a refresh race hurts, and so how strict single-flight refresh has to be.

### A12. The HR platform has a sandbox we can test against
- **Ask:** Is there a sandbox or test tenant? Does it apply the same rate limits and token rotation as production?
- **If wrong:** the fake HR platform becomes the only integration test target. Contract tests run against recorded real responses, and the first customer goes live on a pilot with a small set of learners.

### A13. We have the HR platform's API documentation
- **Why it matters:** the fake HR platform must copy the real request and response shapes (create certification record, list an employee's certification records, error bodies), or the adapter is tested against the wrong contract.
- **Ask:** Which HR platform is it, and can we see its API docs or recorded example responses?
- **If wrong:** model the fake on a typical REST certification endpoint, and keep every vendor-specific shape inside the adapter so it's a single place to fix.

## Our environment

### A8. The ledger and work queue live in the integration service's existing data store
- **Why it matters:** the tokens and course mappings already live there. Keeping the ledger in the same store avoids a consistency problem between two stores, and avoids adding new infrastructure in six weeks.
- **Ask:** What does the existing integration service store its data in?
- **What the design needs either way:** an atomic, conditional claim on a row.
  - **If relational (Postgres/MySQL):** `FOR UPDATE SKIP LOCKED` to claim rows, and a row lock for token refresh. Locks are released automatically if a worker crashes, and customer success can query anything with SQL.
  - **If DynamoDB:** conditional-write leases (`lease_until`, `lease_owner`) to claim rows, and a version check when committing a refreshed token so a worker whose lease expired can't overwrite a newer token. A secondary index of due work per connection. The customer success access patterns (by learner, and by customer and status) must be designed up front as indexes.

### A10. Tokens stay where they're already stored
- **Ask:** Where are tokens stored today (Secrets Manager, or KMS-encrypted columns in the database)? The brief only says "securely".
- **Why it matters:** the token store isn't the lock either way. Single-flight refresh locks on the connection row or lease. If tokens are in Secrets Manager, the refresh spans two systems: save the new secret before releasing the lock, record its `VersionId` on the connection row, and check that version before writing so a stale worker can't overwrite a newer token.
- **If starting fresh:** KMS-encrypted tokens in the same database as the lock, so a refresh is one transaction. Not worth moving them inside six weeks.

### A11. There's existing internal admin tooling that customer success already uses
- **Ask:** Is there an internal admin tool or back-office app we can add a page to?
- **If wrong:** use a Retool/Metabase view over a read replica for the first customer, with plain-English reason text in a lookup table.

## Scope

### A9. Mappings added later trigger the same backfill
- Treated as in scope: "when the feature is first turned on" really means "when a course and certification pair first becomes syncable".
