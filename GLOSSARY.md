# Completion Sync

Sends course completions from our learning platform to the customer's HR platform, where they appear as certification records against the matching employee.

## Language

**Customer**:
An organization that uses our platform to train its people.
_Avoid_: Tenant, account, client

**Connection**:
A customer's authorised link to their HR platform account, including the tokens that keep it working.
_Avoid_: Integration, link, account

**Learner**:
A person training on our platform who has been synced from the HR platform and carries an employee identifier.
_Avoid_: User, student, employee (on our side)

**Employee**:
The HR platform's record of a person, identified by the employee identifier.
_Avoid_: Worker, staff member

**Course mapping**:
An administrator-defined pairing of one of our courses with a certification type on the HR platform.
_Avoid_: Course link, certification mapping

**Mapped course**:
A course that has a course mapping for a given customer.

**Completion**:
One learner finishing one course on a given date, identified by a stable ID from the learning platform.
_Avoid_: Result, attempt, event

**Re-completion**:
A later completion of a course the learner has already completed before; it is a separate completion, not a change to the earlier one.
_Avoid_: Renewal, update

**Syncable completion**:
A completion of a mapped course by a learner whose customer has completion sync turned on.

**Certification record**:
The HR platform's record that an employee holds a certification; a synced completion becomes one of these.
_Avoid_: Qualification, credential

**Backfill**:
Sending the existing syncable completions that predate sync becoming possible: when a customer turns the feature on, or when a new course mapping is added.
_Avoid_: Migration, historical import

**Backfill preview**:
A report of what a backfill would send, and what it couldn't, produced without sending anything; reviewed before the backfill is approved.
_Avoid_: Dry run, simulation

**Revocation**:
An administrator reversing a completion that was granted by mistake.

## Sync status

Where a syncable completion is on its way to the HR platform. Every syncable completion has exactly one sync status, and none is ever dropped.

**Pending**:
Waiting to be sent, either for the first time or to retry after a failure.
_Avoid_: Queued, retrying (retrying is a pending completion that has been attempted before)

**Sending**:
Currently being sent to the HR platform. If a send is interrupted, it's unknown whether the HR platform received it.
_Avoid_: In progress, processing

**Delivered**:
Present on the HR platform as a certification record.
_Avoid_: Synced, sent, complete

**Needs action**:
Can't be delivered until a person fixes something, such as a missing employee or a revoked connection. Always has a reason.
_Avoid_: Failed, dead, error

**Skipped**:
Deliberately not sent because the completion stopped being syncable, such as the customer turning sync off. Always has a reason.
_Avoid_: Cancelled, ignored
