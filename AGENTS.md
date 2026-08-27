# Dinknesh Bingo — Coding Agent Instructions

Before beginning work, read:

```text
.ai/context.md
.ai/architecture.md
.ai/rules.md
.ai/workflow.md
.ai/agents.md
```

Follow the approved AI control system:

```text
Inspect
→ Understand
→ Plan
→ Approval
→ Implement
→ Test
→ Self-review
→ Adversarial review
→ Fix
→ Test again
→ Human review
```

Do not invent requirements or silently change the architecture. Node.js is the business authority, PostgreSQL is the authoritative durable store, and Bot/Mini App clients must not be trusted for financial or game truth. Keep changes scoped, protect secrets, use transactional/idempotent financial and game operations, and report actual test evidence.
