# Dinknesh Bingo — Official AI Development Workflow

All non-trivial work follows this sequence:

```text
INSPECT
   ↓
UNDERSTAND
   ↓
PLAN
   ↓
HUMAN APPROVAL
   ↓
IMPLEMENT
   ↓
TEST
   ↓
SELF-REVIEW
   ↓
ADVERSARIAL REVIEW
   ↓
FIX
   ↓
TEST AGAIN
   ↓
FINAL HUMAN REVIEW
```

## INSPECT

Read the applicable control files, task specification, repository guidance, current code, schema/migrations, tests, and relevant configuration. Identify existing behavior and avoid assuming that a missing file means a missing requirement.

## UNDERSTAND

Translate the approved task into concrete behavior. Identify the authority boundary, affected domain, invariants, dependencies, risks, assumptions, unresolved decisions, and out-of-scope areas.

## PLAN

Write a plan before implementation for any non-trivial task. The plan must include:

- Task identifier and objective.
- Proposed behavior and user-visible impact.
- Affected files/modules/schema.
- Data and state transitions.
- Security, concurrency, and idempotency considerations.
- Risks and assumptions.
- Required tests and evidence.
- Rollback or recovery considerations where relevant.

## HUMAN APPROVAL

The implementing agent must wait for human approval when the task changes architecture, financial behavior, security boundaries, database schema, public API contracts, production behavior, or an unresolved product decision. The plan is not approval. The Human Project Owner has final authority.

Small, clearly scoped, non-architectural changes may follow the project's normal approval convention, but the agent must not infer approval for a risky change.

## IMPLEMENT

Implement only the approved plan. Keep the diff focused. Follow the authority model:

```text
Bot / Mini App → Node.js services → PostgreSQL
```

Persist authoritative state before reporting success. Use transactional and idempotent operations for money and game decisions. Do not silently alter architecture.

## TEST

Run the planned tests and any relevant existing tests. Include:

- Success cases.
- Failure and validation cases.
- Boundary and edge cases.
- Authentication and authorization cases.
- Concurrency/idempotency cases.
- Recovery cases where applicable.

Capture actual commands and outcomes. If a test cannot run, state why and what remains unverified.

## SELF-REVIEW

The implementing agent reviews its own diff against:

- Approved task and plan.
- `.ai/rules.md`.
- `.ai/context.md`.
- `.ai/architecture.md`.
- Security, financial, game, and data invariants.
- Tests and failure handling.

The self-review must check for scope creep, invented behavior, missing validation, unsafe retries, race conditions, secret exposure, and misleading success responses.

## ADVERSARIAL REVIEW

A different AI agent attacks the implementation rather than merely restating it. It looks for logic errors, authorization bypasses, races, duplicate processing, state corruption, invalid Bingo claims, incorrect settlement, weak tests, and omitted edge cases.

The adversarial reviewer reports findings with evidence and severity. It does not silently fix the code or broaden scope.

## FIX

The implementing agent addresses approved findings. Each fix remains scoped. If a finding reveals a genuine architecture conflict, pause and request the Human Project Owner's decision rather than redesigning unilaterally.

## TEST AGAIN

Re-run the full relevant test set after fixes. Include regression tests for each fixed finding. Do not claim completion until the new evidence is available.

## FINAL HUMAN REVIEW

The Human Project Owner reviews the implementation, test evidence, self-review, adversarial report, and any accepted risks. The human decides acceptance, rejection, or further work.

## Completion record

A completed task should leave enough evidence to identify:

- What was changed.
- Which plan and approval authorized it.
- Which tests ran and their results.
- Which review findings were fixed or accepted.
- Which unresolved decisions remain.
