# The completion ledger is the work queue

Each syncable completion becomes a row in a completion ledger, stored in the integration service's existing data store. Delivery workers claim due rows per connection with an atomic conditional claim: `FOR UPDATE SKIP LOCKED` in a relational store, or a conditional-write lease in DynamoDB. The hard constraint is per-connection: 100 requests/minute, 429 with Retry-After, and live completions ahead of backfill. Keeping rate limiting, priority, retry timing and delivery status in one table makes all of them a single query, and gives customer success one place to answer "did it sync?". Peak load is about 2.2 completions/second, well within one database.

## Considered Options

- **SQS FIFO with one message group per connection**: no native per-connection rate limit or priority, so a 429 means holding back a whole message group. Delivery status would be split between the queue and the database. It solves a throughput problem we don't have and not the rate-limit problem we do.
- **SQS standard queue plus the ledger as a status record**: messages for the same connection run in parallel and constantly fight the rate limit.
