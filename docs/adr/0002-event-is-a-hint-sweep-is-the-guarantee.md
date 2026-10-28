# The completion event is a hint; a reconciliation sweep is the guarantee

The legacy platform's completion event is fire-and-forget, so on its own it can't satisfy "no completion may be silently lost". We use the event for low latency, plus a periodic sweep of the legacy internal API for each customer's completions since a watermark. The sweep is the correctness guarantee. Both paths upsert idempotently into the same completion ledger, so duplicates do nothing. Backfill is the same sweep with the watermark set to the beginning, which covers two requirements with one code path.

## Considered Options

- **Event only, with retries on our side**: can't recover an event that was never published or never received.
- **Sweep only**: complete and simple, but latency depends on the poll interval. It's the fallback if the event proves unusable.

## Consequences

- Depends on the legacy internal API being able to list a customer's completions since a timestamp (see `DESIGN-QUESTIONS.md`).
