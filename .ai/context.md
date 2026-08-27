# Dinknesh Bingo — AI Project Context

## Project name

Dinknesh Bingo.

## Project purpose

Dinknesh Bingo is a Telegram-centered Bingo platform. Players register through a Telegram bot, open a Telegram Mini App to view rooms and play Bingo, manage their wallet, inspect history and profile information, and receive notifications. The platform supports deposits, withdrawals, support operations, administration, real-time games, and historical archiving.

## Product overview

The product has two player-facing entry points:

- A Python Telegram Bot for registration, menus, notifications, and bot-based deposit, withdrawal, support, instructions, and referral interactions.
- A Telegram Mini App for the dashboard, game lobby, board selection, live game, results, history, wallet, and profile.

Both entry points use the same Node.js core services. The Node.js backend is the business authority. Supabase PostgreSQL stores authoritative durable state. A private Telegram archive channel is a cold-archive destination for eligible historical game detail, not the primary financial ledger.

## Main user types

- **PLAYER** — registers, joins rooms, selects boards, plays, claims Bingo, views history and wallet data, creates deposits and withdrawals, and opens support tickets.
- **SPECTATOR** — can observe a running game but cannot join the active round, reserve a stake, select a board, or claim Bingo for that round.
- **SUPPORT** — investigates users, games, deposits, withdrawals, and tickets according to explicit permissions.
- **ADMIN** — manages business configuration, users, rooms, financial reviews, support, permissions, reports, and controlled emergency actions according to explicit authorization.

Roles and permissions are loaded and enforced by the backend. Client-supplied roles are never authoritative.

## Main system components

```text
Telegram
├── Python Telegram Bot
└── Telegram Mini App
        │ HTTPS / WebSocket
        ▼
Node.js Core
├── Authentication and authorization
├── HTTP/API layer
├── Wallet service
├── Deposit service
├── Withdrawal service
├── Bingo service / game engine
├── WebSocket / real-time service
├── Notification service
└── Archive service / workers
        │
        ▼
Supabase PostgreSQL
        │
        ▼
Private Telegram archive channel
```

The first deployment keeps the Python Bot separate, while API, WebSocket, Bingo engine, and workers may initially run in one Node.js deployment. The architecture must remain separable by responsibility.

## Main user flows

### Registration

```text
/start
  → find or create user
  → request phone number
  → verify submitted contact belongs to Telegram user
  → verify official-channel membership
  → show main menu
```

Registration creates the account but does not grant money. The initial wallet balance is zero unless a legitimate later deposit or approved bonus exists.

### Play

```text
Telegram Bot
  → PLAY
  → Mini App
  → Telegram authentication
  → server session
  → player dashboard
  → room/stake selection
  → board selection and reservation
  → live game or spectator view
  → result
  → next round
```

### Wallet

The wallet displays total balance, reserved balance, and available balance. Available balance is calculated as:

```text
available_balance = balance - reserved_balance
```

All financial movements also create ledger records.

### History and profile

History is loaded from the backend, not treated as permanent client state. Profile is a read-heavy backend DTO assembled from identity, game history, wallet ledger, referral records, and related data.

## Telegram Bot

The Python Bot is a thin Telegram interface. It handles Telegram updates, keyboards, messages, registration interaction, channel-verification interaction, Mini App launch, temporary conversation state, notifications, and calls to Node.js APIs.

The Bot does **not** own wallet calculations, Bingo rules, prize calculations, winner validation, database business logic, or admin authorization decisions. It must not become a second backend.

Production bot communication uses Telegram webhooks. Bot conversation state is minimal and temporary; authoritative deposit and withdrawal state remains in Node.js/PostgreSQL.

## Telegram Mini App

The Mini App presents:

- Game Home and available rooms.
- Stake selection.
- Board selection for boards 1–600.
- Live game and result screens.
- Spectator view for players arriving during a running round.
- History.
- Wallet, deposits, withdrawals, and transaction details.
- Profile, settings, referrals, and support.

The Mini App may display server-provided values and perform visual interactions, including auto-daub presentation and sound preferences. It never determines authoritative balance, stake, board ownership, game state, called numbers, winning patterns, winners, prizes, or account status.

## Backend

Node.js is the core business authority and exposes domain APIs for authentication, player data, games, deposits, withdrawals, support, bot operations, and admin operations.

The backend owns:

- Telegram identity verification and session creation.
- Role and permission checks.
- Account-status checks.
- Wallet state, reservations, ledger transactions, idempotency, and settlement.
- Room and round lifecycle.
- Board assignment and concurrency protection.
- Official number generation and calling.
- Server-side Bingo validation.
- Winner calculation and prize settlement orchestration.
- Real-time events, snapshots, ordering, and reconnection support.
- Recovery of incomplete work from persisted state.

## Database/state

Supabase provides PostgreSQL. PostgreSQL is the authoritative durable store, and Node.js is the application boundary for business operations.

Logical domains and core tables:

```text
IDENTITY
  users, roles, permissions, role_permissions

FINANCE
  wallets, transactions, deposits, payment_evidence,
  deposit_verifications, withdrawals

BINGO
  rooms, winning_patterns, room_winning_patterns, games,
  boards, game_board_assignments, game_players, game_calls,
  bingo_claims, winners, game_settlement_ledger

PLAYER
  referrals, notifications, support_tickets

SYSTEM
  settings, audit_logs, archive_manifests
```

Important database protections include unique Telegram identity, unique board assignment per game, unique called number per game, stable financial references, transaction-safe wallet mutations, and appropriate constraints against invalid negative or duplicate operations.

Sensitive tables should not be directly writable by clients. Where Supabase client access exists, Row Level Security must enforce ownership. Wallets, transactions, deposits, withdrawals, settlements, admin records, and audit logs should be controlled through Node.js.

## Wallet

Every player has one wallet with a currency, current balance, and reserved balance. A ledger transaction records every financial movement, including deposits, withdrawals, game stakes, refunds, prizes, commissions, referral rewards, and admin adjustments.

Wallet operations must be transaction-safe and idempotent. Concurrent spending must not produce a negative balance or lost update. The wallet service, not the Bot or Mini App, performs reservations, stake finalization, refunds, prize credits, and adjustments.

## Deposit

Bot and Mini App deposits converge on one Deposit Service:

```text
start
  → load enabled payment methods and limits
  → validate method and amount
  → create pending deposit reference
  → show payment instructions
  → receive/parse payment evidence
  → verify amount, provider, destination, transaction ID,
    request status, and duplicate status
  → approve, reject, or manual review
```

Payment evidence preserves raw SMS/message content alongside parsed fields. A valid approval credits the wallet through an idempotent ledger transaction. A mismatch does not credit the wallet. Ambiguous cases go to support. Pending requests expire so old requests cannot match unrelated future payments.

The SMS gateway is evidence transport, not financial authority.

## Withdrawal

Bot and Mini App withdrawals converge on one Withdrawal Service:

```text
request
  → validate active account, amount, limits, and destination
  → reserve funds
  → create withdrawal request
  → confirm to player
  → risk/provider processing
  → complete, fail, reject, cancel, or manual review
```

Only enabled methods are shown. The architecture favors verified player-owned destination accounts. Funds are reserved before processing and released on failure, rejection, or cancellation. Confirmation evidence should come from an official provider result, trusted payment system, or controlled SMS gateway; SMS alone should not be treated as the sole authority for payout completion.

The final approval/processing policy is controlled by authorized Admin/Support workflow, not assumed to be automatic.

## Bingo game

The Bingo engine runs in Node.js. A room represents a stake/configuration, and one room has one active round at a time. There are 600 fixed deterministic 5×5 boards. Board numbers are unique only within a game, so the same board can be used in a later game.

The server enforces a maximum of two boards per player. Board selection atomically coordinates board assignment and wallet reservation. A database uniqueness constraint is the final protection against two users claiming the same board in the same game.

Standard 75-ball column ranges are:

```text
B = 1–15, I = 16–30, N = 31–45, G = 46–60, O = 61–75
```

The server creates a non-repeating authoritative sequence. The center space is free and counts as marked.

## Game rooms

Rooms expose admin-controlled business configuration such as name, stake, enabled state, player limit, board limit, per-player board limit, selection duration, result duration, call interval, winning patterns, prize percentage, platform fee percentage, and minimum players.

Each active game receives a configuration snapshot. Later configuration changes apply to a later game, not retroactively to a running game.

Players arriving during selection may participate. Players arriving during gameplay become spectators and wait for the next board-selection phase.

## Authentication

The Mini App authenticates with Telegram init data through Node.js, which verifies Telegram data and creates an application session. The Bot also routes identity and registration actions through Node.js.

Authentication occurs before WebSocket access. Sessions, Telegram identity, account status, and role are checked server-side. Authentication is not authorization.

## Authorization

Authorization is role-based plus permission-based. The backend checks the exact permission for each operation, rather than trusting a UI menu or a client-supplied role.

Players may perform player operations on their own account. Support access is limited to assigned support capabilities. Admin has the highest application-level privilege but cannot change secrets, bypass security controls, silently rewrite completed history, or erase audit/financial records.

Dangerous admin actions require confirmation, a reason, and an audit log. Corrections create compensating records rather than rewriting history.

## Real-time communication

WebSocket is the live transport. Node.js is the real-time authority; Supabase Realtime is not the authoritative game engine.

HTTP/API is used for commands and ordinary reads. WebSocket is used for public game events and targeted private events.

Public examples:

```text
GAME_STARTED, NUMBER_CALLED, PLAYER_COUNT_UPDATED,
BINGO_CLAIMED, WINNER_DECLARED, GAME_FINISHED, NEXT_ROUND
```

Private examples:

```text
BOARD_ASSIGNED, BALANCE_UPDATED, YOUR_BINGO_AVAILABLE
```

Events carry sequence information. Reconnection authenticates again, identifies the authorized room/game, obtains a current snapshot, replays missing events where available, and replaces stale client state.

## Security principles

- The server is authoritative for identity, permissions, wallet, game state, official calls, boards, winners, prizes, and configuration.
- Never expose secrets or trust client-supplied financial values, roles, game state, called numbers, or winning patterns.
- Enforce authentication and authorization server-side.
- Use database constraints and transactions for wallet, board, game, and idempotency protections.
- Make deposits, withdrawals, Bingo claims, settlement, and archive jobs safely retryable.
- Use secure server-side randomness; the browser cannot generate the authoritative sequence.
- Use input validation, rate limiting, anti-spam controls, secure logging, audit logs, RLS where applicable, and protected WebSocket authorization.
- Fail closed for critical operations when the database or required authority is unavailable.
- Preserve evidence and financial history.

## Important business rules

- Registration does not grant money.
- A user must pass phone ownership verification and required channel membership checks.
- One Telegram account maps to one application user.
- A player can select at most two boards in a game.
- A board can belong to only one player in a particular game.
- Selection closes according to the server timestamp, not a browser timer.
- Locked games accept no new boards, players, stake changes, or board changes.
- New arrivals during a running game are spectators.
- Official Bingo calls are server-generated, non-repeating, persisted, and broadcast to all authorized room viewers.
- Auto-daub is a client presentation option; it never changes authoritative game truth.
- Winning patterns are enabled by room/game configuration and validated server-side.
- A valid board can use a three-second claim window with automatic claim at zero.
- Multiple winners are supported; the configured winner policy determines allocation.
- The Bingo engine calculates the result; the Wallet Service performs financial settlement.
- A settled game is immutable.
- The platform fee is configured; the architecture source describes a 10% fee example.

## Important constraints

- Python Bot is not a second backend.
- Mini App clients do not directly mutate privileged financial or game state.
- Node.js owns the game engine and WebSocket authority.
- Supabase/PostgreSQL stores durable authoritative state.
- The first deployment should avoid unnecessary microservice complexity and use one authoritative Bingo worker process.
- Future multiple game workers require a room ownership/lease mechanism.
- Cold archive is secondary storage, not a replacement for the financial ledger.
- Archive completed game detail only after creation, verification, and manifest marking.
- Never delete financial/accounting records to solve storage pressure.

## Items not yet defined

These items require an explicit product or architecture decision before implementation where they affect behavior:

- Exact rules for releasing a board reservation when a player leaves before lock.
- Exact policy for collecting simultaneous Bingo claims and ending a round.
- Exact winner policy and rounding behavior for multiple winners.
- Final enabled winning patterns for each room.
- Final provider integrations, payment APIs, SMS gateway contract, and evidence formats.
- Exact withdrawal approval and manual-review policy.
- Exact minimum-player and game-cancellation/refund behavior.
- Final board storage representation and board-generation/verification process.
- Exact session/token implementation and key rotation details.
- Final rate limits, audit verbosity, archive retention threshold, and storage warning values.
- Final deployment host, domains, scaling mechanism, room-worker ownership mechanism, and operational monitoring choices.
- Localization language files and final player-facing wording.
