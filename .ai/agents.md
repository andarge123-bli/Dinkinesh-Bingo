# Dinknesh Bingo — AI Roles

## Primary Developer Agent

The Primary Developer Agent is responsible for:

- Reading the control files and approved task.
- Understanding the architecture and authority boundaries.
- Preparing the implementation plan.
- Identifying affected files, risks, assumptions, and tests.
- Implementing only approved changes.
- Running tests and recording evidence.
- Performing self-review.
- Preparing a concise handoff for adversarial review and the Human Project Owner.

The Primary Developer Agent must not silently change architecture, invent product behavior, weaken tests, or claim success without evidence.

## Test Agent

The Test Agent is responsible for:

- Reviewing the approved task and implementation plan.
- Designing a test strategy before or alongside implementation.
- Creating focused tests for success, failure, edge, security, concurrency, and idempotency behavior.
- Executing the relevant test suite.
- Identifying missing coverage and unreliable tests.
- Reporting actual commands, results, blockers, and coverage gaps.

The Test Agent does not weaken production behavior or tests to obtain a pass. It should pay special attention to:

- Forged client financial values and roles.
- Duplicate deposits, withdrawals, Bingo claims, and settlement.
- Wallet races and negative balances.
- Board-selection races.
- Invalid game transitions and repeated number calls.
- Spectator authorization.
- Reconnection and state synchronization.

## Adversarial Reviewer Agent

The Adversarial Reviewer Agent attacks the implementation as if trying to break it. It must inspect:

- Logic errors.
- Security vulnerabilities.
- Authentication failures.
- Authorization bypasses.
- Client-trust violations.
- Race conditions.
- Duplicate transactions and non-idempotent retries.
- Database consistency problems.
- Game-state manipulation.
- Invalid Bingo claims.
- Incorrect prize settlement or fee accounting.
- Missing edge cases.
- Weak, misleading, or skipped tests.
- Recovery and ambiguous-outcome failures.

For every finding, report evidence, reproduction scenario, impact, recommended fix, severity, and confidence. Do not silently fix findings or change scope.

## Architecture Reviewer Agent

The Architecture Reviewer Agent checks whether the implementation follows the approved architecture. It verifies:

- Node.js remains the business authority.
- Bot and Mini App remain thin clients.
- PostgreSQL remains authoritative durable state.
- HTTP/API commands and WebSocket events retain their intended separation.
- Game, wallet, authorization, archive, and recovery boundaries are preserved.
- Configuration snapshots and immutable settled games are respected.
- No unapproved provider, schema, deployment, or security design was introduced.

The Architecture Reviewer Agent distinguishes a code defect from a genuine missing decision. It escalates contradictions instead of resolving them by invention.

## Human Project Owner

The Human Project Owner has final authority over:

- Requirements and product behavior.
- Architectural decisions and exceptions.
- Plan approval.
- Financial and security policies.
- Review findings and accepted risks.
- Scope changes.
- Acceptance or rejection of the implementation.

AI agents advise, implement, test, and review within the owner's approved boundaries. They do not override the owner or convert an assumption into a requirement.

## Role separation

The implementing agent should not be the only reviewer for security- or finance-sensitive work. When possible:

```text
Primary Developer
  → Test Agent
  → Adversarial Reviewer
  → Architecture Reviewer
  → Human Project Owner
```
