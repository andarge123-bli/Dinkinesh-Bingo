# Dinknesh Bingo — AI Agent Rules

These rules apply to every AI agent working on the project.

## General

1. Read `AGENTS.md`, `CLAUDE.md` or `GEMINI.md` as applicable, and all `.ai/` control files before working.
2. Treat `.ai/context.md` and `.ai/architecture.md` as the approved project specification.
3. Do not invent requirements, integrations, security exceptions, or technical details absent from the architecture.
4. Preserve existing architectural decisions unless the Human Project Owner explicitly approves a change.
5. Do not redesign, simplify away, or reinterpret a business rule silently.
6. Keep changes scoped to the approved task. Do not make unrelated cleanup or formatting changes.
7. Do not change control files to justify an implementation that was not approved.
8. If the architecture and task conflict, stop at the conflict, document it, and request a decision.

## Planning

1. Every non-trivial task requires a written plan before implementation.
2. The plan must state the task ID or source request.
3. Identify affected files and modules.
4. Identify dependencies, risks, assumptions, and unresolved architecture decisions.
5. Identify required unit, integration, failure, edge-case, and security tests.
6. Distinguish facts from assumptions.
7. Do not begin implementation until the workflow's approval requirement is satisfied.
8. If a task changes a financial, game, authorization, schema, or security invariant, call that out explicitly.

## Implementation

1. Implement only the approved scope and plan.
2. Follow the authority boundaries: Node.js owns business truth; Bot and Mini App are clients.
3. Never trust client-provided financial values, roles, game state, called numbers, boards, winners, or prizes.
4. Put financial operations behind transactional, idempotent service logic.
5. Use database constraints and concurrency protection for wallet and board operations.
6. Persist authoritative state before publishing success or dependent events.
7. Enforce valid game-state transitions server-side.
8. Keep game configuration snapshot-based; do not change active-round rules mid-game.
9. Keep secrets in the approved secrets/environment mechanism. Never hard-code or print them.
10. Do not modify migrations, schemas, financial history, or production-facing configuration without explicit task scope.
11. Do not add dependencies or external services unless the task and architecture approve them.
12. Preserve backward compatibility where the task does not authorize a breaking change.

## Testing

1. Test every new behavior with actual executable evidence.
2. Test expected success paths and failure paths.
3. Test edge cases and boundary values.
4. Test concurrency and idempotency for financial/game operations.
5. Test server-side authentication and authorization, including forged client values.
6. Test invalid game transitions, duplicate calls, duplicate claims, duplicate payment evidence, and duplicate withdrawals.
7. Test board-selection races and maximum-board enforcement.
8. Test recovery after process, connection, WebSocket, database, provider, and archive failures where relevant.
9. Do not weaken, delete, skip, or rewrite tests merely to make them pass.
10. Do not claim a test passed without running it and recording the command/result.
11. Record known untested areas and environmental blockers honestly.

## Security

1. Never expose credentials, tokens, signing keys, database credentials, or other secrets.
2. Never accept a client-provided balance, available balance, role, permission, stake, prize, winner, official call, or game state as truth.
3. Verify Telegram authentication and session data server-side.
4. Enforce authorization server-side on every protected operation.
5. Protect financial operations against duplicate processing, races, timeouts, and ambiguous outcomes.
6. Protect Bingo against manipulated boards, calls, patterns, claims, timers, and winners.
7. Fail closed when authoritative database state cannot be confirmed.
8. Use audit logs for admin/support financial actions, configuration changes, emergency actions, and compensating records.
9. Do not log raw payment evidence or personal data beyond the approved need.
10. Do not use archive storage as a substitute for the primary financial ledger.

## Review and communication

1. Separate implementation findings from architecture change proposals.
2. Mark assumptions and unresolved decisions explicitly.
3. Report evidence, not confidence alone.
4. Do not hide unrelated pre-existing failures.
5. A self-review is mandatory before handoff.
6. A different agent should perform adversarial review when the workflow calls for it.
7. The Human Project Owner has final authority over requirements, architecture, plan approval, review findings, and acceptance.
