# Dinknesh Bingo — Approved Architecture

This document is the AI-facing architecture summary derived from the supplied Dinknesh Bingo architecture source. It preserves the source decisions and records unresolved items rather than silently designing them.

## System Overview

Dinknesh Bingo is a Telegram bot plus Telegram Mini App product backed by a Node.js core and Supabase PostgreSQL.

```text
Telegram
├── Python Bot
└── Mini App
        │
        ├── HTTPS commands and reads
        └── WebSocket live updates
        ▼
Node.js Core
├── Authentication / Authorization
├── Wallet / Deposit / Withdrawal
├── Bingo Engine
├── Notification and Support services
└── Background workers
        ▼
Supabase PostgreSQL
        ▼
Private Telegram cold archive
```

### Authority boundaries

- Telegram supplies identity and transport for the Bot/Mini App entry points.
- The Python Bot is a thin Telegram interface.
- The Mini App is a presentation/client-interaction layer.
- Node.js is the sole application business authority.
- PostgreSQL stores authoritative durable state.
- The archive is secondary historical storage.

No client may decide or directly mutate balance, reservations, winner status, prizes, official calls, game transitions, or withdrawal/deposit approval.

## Telegram Bot

The Python Bot handles:

- Telegram updates and production webhooks.
- Keyboards, messages, and notifications.
- `/start`, registration interaction, phone collection, and channel-verification interaction.
- Main menu and Mini App launch.
- Deposit, withdrawal, support, instructions, and referral conversations.
- Minimal temporary conversation state.
- Calls to Node.js APIs.

The Bot does not implement:

- Wallet calculations or ledger authority.
- Bingo rules or number generation.
- Winner validation or prize calculations.
- Database business logic.
- Admin authorization.

### Registration

```text
/start
  → extract Telegram identity
  → ask Node.js whether user exists
  → create/find basic account
  → request Telegram contact
  → server verifies contact ownership
  → check official-channel membership through Telegram
  → complete registration
  → show role-appropriate menu
```

Registration must not credit money. Existing users continue after server-side status checks. Banned or otherwise restricted accounts are handled by Node.js decisions.

### Bot-to-core communication

Bot calls Node.js over HTTP/HTTPS. Node.js returns authoritative status and data. Even Admin and Support menu entries are only UI conveniences; every operation is authorized again by Node.js.

## Player Registration

The initial user record includes Telegram identity, profile names, phone number, role, status, and timestamps. `telegram_id` is unique.

Phone verification exists to prevent attaching another person's contact to the registering account. A button press is not proof of channel subscription; the backend uses Telegram's membership response.

Account states include active, suspended, banned, and deleted-style historical retention. Important financial and audit records remain even when an account is no longer active.

## Telegram Mini App

The Mini App has four primary areas:

```text
GAME      current/next Bingo participation
HISTORY   past game summaries and details
WALLET    balance, transactions, deposits, withdrawals
PROFILE   identity, statistics, referrals, settings, support
```

Game screens include Game Home, Stake/Room, Board Selection, Live Game, Game Result, and Spectator View. Wallet screens include Wallet Home, Deposit, Deposit Status, Withdraw, Withdrawal Status, and Transaction Details.

After Telegram authentication, the Mini App requests a player dashboard containing player data, wallet data, available rooms, and active-game information. Room/stake values come from the server and may be controlled by Admin.

The Mini App may:

- Render server state.
- Send commands through HTTP/API.
- Subscribe to authorized WebSocket updates.
- Auto-mark or visually highlight board numbers.
- Store local presentation preferences such as sound.

It may not authoritatively set or override server state.

## Authentication

The Mini App obtains Telegram init data and sends it to Node.js. Node.js verifies the data, finds the user, checks account status, and creates an application session.

The Bot also routes identity operations to Node.js. WebSocket access happens only after authentication and session validation.

Authentication establishes identity. Every protected operation still performs authorization and account-status checks.

## Authorization

Use roles plus granular permissions:

```text
PLAYER, SUPPORT, ADMIN
```

Representative permissions include:

```text
view_game
view_history
create_deposit
create_withdrawal
view_support
manage_support
ban_user
approve_deposit
approve_withdrawal
adjust_balance
manage_rooms
manage_settings
```

The backend checks the exact permission for the exact operation. Client roles, menu visibility, and request fields are not authorization.

Admin can manage business configuration, users, rooms, reviews, support, permissions, and reports. Support has a narrower investigation/support role. Support balance adjustments are separately configurable and default to off in the source architecture.

No role may change secrets, silently rewrite completed games or financial history, or bypass security controls.

## Player Dashboard

The dashboard aggregates:

- Player identity/status.
- Wallet balances.
- Enabled/available rooms.
- Stake values and player counts.
- Active game context.

The backend owns all values. The frontend is not trusted to report balance, stake, room availability, player count, or game state.

## Game Lobby

The lobby displays rooms configured and enabled by Admin. A room represents a stake and operational configuration. Joining a room checks:

```text
user exists
account active
room active
current game accepts players
available balance >= stake
```

Joining a running game does not make the player a participant in that running round; the player becomes a spectator according to the active-round rules.

## Stake Selection

The player selects a configured room/stake. The backend validates the current configuration and available balance. Insufficient funds are rejected server-side and may direct the player to Deposit.

The same domain service is used whether the action originated in the Mini App or Bot.

## Board Selection

Each game exposes boards 1 through 600. A player can select no more than two boards. The backend validates:

```text
game is still accepting selection
board exists
board is available in this game
player has fewer than two boards
player has enough available funds
wallet reservation can be made
```

The selection command is an HTTP/API request. A successful assignment is broadcast so other clients mark that board occupied. A failed request must not be represented as successful client state.

## Board Reservation

Board assignment and stake reservation must be coordinated atomically.

```text
available balance  → reservation
available board    → assignment
```

The database must enforce:

```text
UNIQUE(game_id, board_id)
```

Two simultaneous attempts for the same board produce one success and one conflict. The exact pre-lock leave/release policy remains an item requiring decision.

At selection close, the final participant and board set is frozen. No new participant, board change, stake change, or board selection is accepted.

## Wallet

Wallet state includes:

```text
balance
reserved_balance
currency
```

Available balance is derived:

```text
balance - reserved_balance
```

The transaction ledger records every movement, including:

```text
DEPOSIT
WITHDRAWAL
GAME_STAKE
GAME_REFUND
PRIZE
COMMISSION
REFERRAL_REWARD
ADMIN_ADJUSTMENT
```

Wallet operations use database transactions, concurrency protection, stable business references, and idempotent processing.

## Deposit

Bot and Mini App use one Deposit Service.

```text
start deposit
  → load enabled payment methods and limits
  → choose method
  → validate amount
  → create request/reference before payment
  → show destination and instructions
  → receive payment evidence
  → parse and retain raw evidence
  → verify request/evidence match
  → APPROVE / REJECT / MANUAL_REVIEW
```

Deposit records include the player, wallet, method, amount, destination, status, reference, provider transaction ID, expiry, and timestamps. Evidence and verification attempts are separate auditable records.

An approval credits the wallet once. A duplicate provider transaction/reference never credits again. A mismatch or ambiguous case does not guess; it rejects or routes to authorized review according to the final policy. Expired requests cannot match old evidence.

The SMS gateway forwards evidence and is not the financial authority.

## Withdrawal

Bot and Mini App use one Withdrawal Service.

```text
request
  → validate user/status/amount/limits/destination
  → reserve funds
  → create withdrawal record
  → show confirmation
  → risk/provider checks
  → payout or manual review
  → confirm outcome
  → complete, fail, reject, cancel, or release reservation
```

Only enabled methods are available. The architecture favors verified player-owned destination accounts. Funds remain reserved while active. Failure, rejection, or cancellation releases the reservation. Duplicate requests and duplicate provider evidence are idempotent.

The source architecture calls for authorized Admin/Support approval for the final process rather than assuming unrestricted automatic payout. Exact approval policy and provider contract remain to be finalized.

## Bingo Game Engine

Node.js owns:

- Room and game lifecycle.
- Selection timer and lock.
- Authoritative non-repeating number sequence.
- Calling interval.
- Winning detection and claim processing.
- Result, settlement, and next-round creation.

It must not depend on a browser timer, browser JavaScript, a Telegram client, or the presence of any particular WebSocket connection.

### Fixed boards

There are 600 fixed deterministic 5×5 boards. A board is reference data and can be reused in later games. Ownership is represented separately by game assignment.

Standard 75-ball mapping:

```text
B = 1–15
I = 16–30
N = 31–45
G = 46–60
O = 61–75
```

The center square is free and automatically marked for validation.

## Game State Lifecycle

The authoritative lifecycle is:

```text
SELECTION
    ↓
LOCKED
    ↓
RUNNING
    ↓
RESULT
    ↓
SETTLEMENT
    ↓
FINISHED
    ↓
next SELECTION
```

The game state is persisted. Valid transitions are enforced server-side. A finished game cannot return to running.

### Selection

The server stores selection start/end timestamps. The frontend countdown is visual only. At the authoritative end time, the server locks assignments.

### Locked

The participant set and board assignments are immutable for the round. New arrivals observe the running game as spectators.

### Running

The engine initializes the game ID, authoritative sequence/seed information, called-number state, winning-rule configuration, and participants. It calls numbers sequentially and persists every call.

### Result, settlement, finished

After valid winners are determined, the system settles stakes, calculates the prize allocation, credits winners, closes/releases reservations as applicable, writes ledger records, publishes the result, and marks the game immutable.

### Next round

The room creates the next game and re-enters selection. Existing and new players may participate in the new selection phase.

## Number Calling

The server generates secure randomness, shuffles the valid 1–75 sequence, and calls each number at most once. The browser cannot generate or influence official numbers.

Each official call is permanent game evidence with game, sequence, number, column, label, and timestamp. The event is broadcast to all authorized room participants and spectators.

## Real-Time Events

WebSocket is used for live updates; HTTP/API is used for commands and ordinary reads.

Public event examples:

```text
GAME_STARTED
NUMBER_CALLED
PLAYER_COUNT_UPDATED
BOARD_RESERVED
BINGO_CLAIMED
WINNER_DECLARED
GAME_FINISHED
NEXT_ROUND
```

Private event examples:

```text
BOARD_ASSIGNED
BALANCE_UPDATED
YOUR_BINGO_AVAILABLE
```

Events include ordering/sequence information. A room channel is logically associated with a game. Spectators receive public events but cannot perform participant-only actions.

## Auto-Daub

When `NUMBER_CALLED` arrives, the Mini App checks whether the called number exists on each displayed board:

```text
server calls number
  → client visually marks/highlights matching cell
```

Auto-daub on/off is a presentation preference. The server's called-number set, not the client's marked cells, determines a win.

## Bingo Validation

After each official call, the server evaluates participating boards against the enabled winning patterns. Supported pattern types in the architecture are:

```text
COLUMN_B
COLUMN_I
COLUMN_N
COLUMN_G
COLUMN_O
ROW
DIAGONAL
FOUR_CORNERS
```

Not all patterns are active in every room. Enabled patterns come from the room/game configuration snapshot.

For a claim, the server independently verifies:

```text
authenticated user
game exists and is RUNNING/claimable
player participates
board belongs to player for this game
called numbers are official
pattern is enabled
board satisfies the pattern
claim is not already processed
```

Client-supplied winning pattern, board truth, called numbers, or prize is ignored as authoritative input.

## Multiple Winners

The model supports multiple valid winners for one winning event. The winner relation is one-to-many, not a single `winner_id`.

The architecture describes a three-second claim window after a board qualifies and automatic claim at zero to cover slow or disconnected clients. The exact simultaneous-claim collection and winner policy must be finalized before implementation.

## Prize Settlement

The Bingo engine determines the winning result and intended allocation. The Wallet Service performs the financial operations transactionally.

Settlement includes:

```text
finalize stakes
calculate prize pool and platform fee
allocate winner payouts
credit winner wallets
close/release reservations
write ledger/settlement records
```

The architecture contains a 10% platform-fee example and equal-split examples. Exact configured fee, payout rounding, and winner-policy rules must remain configuration/product decisions.

## History

Recent player history is kept as compact summaries sufficient for the History screen: game, player, result, stake, winnings, board numbers, winning pattern, winner count, date, and archive reference.

Detailed completed game records may move to cold archive after the archive pipeline creates, compresses, hashes, uploads, verifies, and manifests them. The financial ledger and critical evidence remain in PostgreSQL.

## Spectators

Players entering during a running game are recorded as spectators where participation/session tracking requires it. Spectators receive public events and the result, but:

```text
cannot select a board
cannot reserve a stake
cannot claim Bingo
cannot receive a prize for that round
```

They can join normally when the next selection phase begins.

## New Player During Active Game

```text
arrives during SELECTION
  → can join/select available boards

arrives during RUNNING
  → spectator
  → sees current snapshot and subsequent public events
  → sees result
  → waits for next SELECTION
```

## Database Concepts

Core relationships:

- `users` own wallets, deposits, withdrawals, referrals, tickets, and history summaries.
- `rooms` contain games.
- `games` contain players, board assignments, calls, claims, winners, and settlement.
- `boards` are fixed reference data.
- `game_board_assignments` connect a board and user within one game.
- `transactions` connect financial movements to business references.
- `settings` hold domain configuration.
- `audit_logs` preserve sensitive actions and changes.

Critical constraints include:

```text
UNIQUE(users.telegram_id)
UNIQUE(game_board_assignments.game_id, game_board_assignments.board_id)
UNIQUE(game_calls.game_id, game_calls.sequence)
UNIQUE(game_calls.game_id, game_calls.number)
```

Wallet mutations require row-level locking or equivalent transaction isolation. RLS applies to any direct client-exposed Supabase access; privileged financial/game/admin tables remain behind Node.js.

## Security Rules

1. Verify Telegram identity server-side.
2. Authorize every protected operation server-side.
3. Check account status on access and mutation.
4. Ignore client-supplied balances, roles, stakes, board ownership, game state, calls, winners, and prizes.
5. Use stable references and unique constraints for financial idempotency.
6. Prevent double spending through transactional wallet operations.
7. Protect board selection with atomic assignment and a database uniqueness constraint.
8. Generate and persist the official Bingo sequence only on the server.
9. Validate Bingo claims from authoritative server data.
10. Snapshot game configuration; never change active-round rules mid-game.
11. Protect WebSocket authentication, room authorization, event ordering, and reconnection.
12. Rate-limit authentication, claims, withdrawals, registration, and other abuse-prone actions.
13. Never log secrets or sensitive data unnecessarily.
14. Audit admin, support, configuration, financial, and emergency actions.
15. Fail closed when critical database authority is unavailable.
16. Back up critical financial/game/audit data; do not treat Telegram archive as the only backup.

## Failure and Recovery

Recover from persisted state, not process memory:

```text
load incomplete state
  → inspect status/idempotency
  → resume or safely finalize
  → broadcast current state
```

Player disconnects do not stop a game. Reconnection authenticates, gets a current snapshot, determines the last sequence, replays missing events where possible, and replaces stale client state.

Database failures fail closed for critical operations. Ambiguous financial timeouts are resolved by checking the stable operation reference rather than blindly retrying.

Workers retry temporary failures with backoff but do not retry permanent business errors such as insufficient balance, invalid claims, banned access, or duplicate boards.

## Archive and Cold Storage

The archive worker handles eligible old game detail:

```text
find old games
  → build compact archive
  → gzip
  → compute SHA-256
  → upload to private Telegram archive channel
  → verify
  → write/update archive manifest
  → apply retention only after verification
```

PostgreSQL retains users, wallets, ledger, deposits, withdrawals, payment verification, active/recent games, compact history, settings, permissions, manifests, and critical audit data. Old game detail, board assignments, calls, claims, winners, configuration snapshots, and result details may be archived.

Storage monitoring must operate below the hard database quota. The source architecture gives illustrative thresholds, but final values remain configurable. Archive failures preserve source records and are retried. Archives are idempotent and manifests are not casually deleted.

## Deployment Architecture

Initial deployment:

```text
Python Bot        independent deployment
Node.js Core      API + WebSocket + Bingo + workers
Supabase          managed PostgreSQL
Archive Worker    initially inside/alongside Node.js responsibility
SMS Gateway       receives/forwards payment evidence
```

Node.js logical modules:

```text
API Server
Authentication Service
Authorization
Wallet Service
Deposit Service
Withdrawal Service
Bingo Service
Realtime Service
Notification Service
Archive Service
```

The 24/7 scheduler isolates room state logically. Version 1 uses one authoritative Bingo worker process. If multiple Node.js instances or game workers are introduced, a room ownership/lease mechanism is mandatory.

## Important Architectural Principles

- Preserve one business authority: Node.js.
- Separate commands from events: HTTP/API commands, WebSocket updates.
- Persist before claiming success.
- Use database constraints as final concurrency protection.
- Reserve first and settle later for board entries and withdrawals.
- Make retries safe through idempotency.
- Keep live game state small and recoverable.
- Do not persist UI/realtime noise unnecessarily.
- Keep critical financial history in PostgreSQL permanently.
- Change configurable business rules through authorized APIs and audit changes.
- Treat the architecture as frozen until a genuine contradiction or explicit owner-approved change is found.
