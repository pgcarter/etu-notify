# Problem statement

## Context

We have a learning platform. Customers use it to train their people. Many customers keep their employee records in an HR platform.

Some of the integration exists already:
- A customer can make a connection to their HR platform.
- Each learner has an employee identifier from the HR platform.
- Administrators can make course mappings between our courses and HR certification types.

## The problem

When a learner completes a mapped course, the HR platform must show it. The completion must become a certification record on the matching employee. **We don't send anything to the HR platform yet.**

## What makes it hard

1. **The event can be lost.** The legacy platform sends an event when a learner completes a course. It doesn't send the event again if it fails. But no completion may be lost without someone knowing.
2. **The rate limit is low.** The HR platform accepts 100 requests per minute for each connection. In compliance season, one large customer can have more completions than that limit permits, so completions will wait.
3. **Retries can make duplicates.** A request can time out after the HR platform has already saved the record. If we send it again, we must not make a second certification record.
4. **The tokens are fragile.** The refresh token changes each time we use it. If two workers refresh at the same time, the connection can stop working.
5. **History is necessary.** When a customer turns on sync, we must also send their earlier completions (backfill).
6. **Customer success must be able to see the result.** They must be able to answer "did this completion sync?" without an engineer.

## Limits

- Three engineers.
- The first customer is live in 6 weeks.
- Compliance season starts in 10 weeks.
- We don't change the legacy platform.

## In one line

Send each syncable completion to the HR platform once, and only once. Lose none. Stay inside the rate limit. Show the status of each completion.
