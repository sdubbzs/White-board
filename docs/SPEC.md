# Household Task Board — Build Spec v1

## Product summary
A mobile app for people living together (target: younger couples) to fairly split household duties. The core surface is a **shared board** — a lightweight whiteboard where either person can post a task, assign it, or claim it. Completed tasks earn points; points feed a **weekly scoreboard** and unlock **rewards** the household defines itself.

Tone: playful, joint effort. Explicitly *not* a surveillance or nagging tool. Both members see the same board and the same numbers.

## Stack
- **App:** React Native via Expo (iOS + Android)
- **Backend:** Node + Express
- **DB:** Postgres
- **Realtime:** Socket.IO
- **Auth:** email + password, JWT access token + refresh token
- **Storage (phase 6):** S3-compatible bucket for proof photos

Language is JavaScript across the stack.

## Code conventions
- One module per concern; no multi-purpose files.
- One file per model, one file per route group, one file per API resource on the client.
- Named exports, no default exports except React components.
- All DB access goes through model modules — routes never write SQL.
- Migrations are numbered and forward-only.
- Every endpoint validates input at the route boundary before touching a model.
- `.env` for secrets; commit `.env.example` only.
- README explains setup, migrations, and how to run both halves.

## Data model

```
User
  id              uuid pk
  email           text unique not null
  password_hash   text not null
  display_name    text not null
  created_at      timestamptz default now()

Household
  id              uuid pk
  name            text not null
  created_at      timestamptz default now()

Membership
  id              uuid pk
  user_id         uuid fk -> User
  household_id    uuid fk -> Household
  role            text check (role in ('owner','member'))
  joined_at       timestamptz default now()
  unique (user_id, household_id)

Invite
  id              uuid pk
  household_id    uuid fk -> Household
  token           text unique not null      -- url-safe random
  created_by      uuid fk -> User
  expires_at      timestamptz not null
  accepted_at     timestamptz null
  accepted_by     uuid fk -> User null

Task
  id              uuid pk
  household_id    uuid fk -> Household
  title           text not null
  notes           text null
  points          int not null check (points in (5,10,15))
  assignee_id     uuid fk -> User null       -- null = unclaimed, on the board
  status          text check (status in ('open','claimed','done'))
  due_at          timestamptz null
  created_by      uuid fk -> User
  created_at      timestamptz default now()
  completed_at    timestamptz null
  completed_by    uuid fk -> User null
  proof_photo_url text null

ScorePeriod
  id              uuid pk
  household_id    uuid fk -> Household
  starts_at       timestamptz not null
  ends_at         timestamptz not null
  unique (household_id, starts_at)

Reward
  id              uuid pk
  household_id    uuid fk -> Household
  title           text not null
  cost_points     int not null
  created_by      uuid fk -> User
  active          boolean default true

Redemption
  id              uuid pk
  reward_id       uuid fk -> Reward
  redeemed_by     uuid fk -> User
  household_id    uuid fk -> Household
  redeemed_at     timestamptz default now()
  points_spent    int not null
```

**Scores are never stored.** A member's score for a period = sum of `points` on tasks where `completed_by = user`, `status = 'done'`, and `completed_at` falls inside the period. Redemptions are tracked separately so the leaderboard stays honest.

Points tiers: **5** (quick — dishes, bins), **10** (standard — laundry, hoovering), **15** (heavy — deep clean, big shop).

## API

**Auth**
```
POST   /auth/register        { email, password, display_name }
POST   /auth/login           { email, password }
POST   /auth/refresh         { refresh_token }
POST   /auth/logout
GET    /me
```

**Households & invites**
```
POST   /households                  { name }           -> creates + owner membership
GET    /households/:id              -> household, members
POST   /households/:id/invites      -> { url, expires_at }
GET    /invites/:token              -> household preview (unauthenticated)
POST   /invites/:token/accept       -> joins caller, marks invite accepted
DELETE /households/:id/members/:uid
```

Invite link format: `https://<host>/join/<token>`, 7-day expiry, single use. Deep-links into the app; falls back to a web page with store links.

**Tasks**
```
GET    /households/:id/tasks?status=&assignee=
POST   /households/:id/tasks        { title, notes, points, assignee_id?, due_at? }
PATCH  /tasks/:id                   { title?, notes?, points?, due_at? }
POST   /tasks/:id/claim             -> assignee = caller, status = 'claimed'
POST   /tasks/:id/unclaim
POST   /tasks/:id/complete          { proof_photo_url? }
POST   /tasks/:id/reopen
DELETE /tasks/:id
```

**Scoreboard**
```
GET    /households/:id/scoreboard?period=current|previous
       -> { period, entries: [{ user_id, display_name, points, tasks_done }] }
```

**Rewards**
```
GET    /households/:id/rewards
POST   /households/:id/rewards      { title, cost_points }
PATCH  /rewards/:id                 { title?, cost_points?, active? }
POST   /rewards/:id/redeem
GET    /households/:id/redemptions
```

## Realtime
Socket.IO namespace `/board`. On connect, client authenticates with its access token and joins room `household:<id>`.

Server emits to the room on every mutation:
```
task:created    { task }
task:updated    { task }
task:deleted    { task_id }
scoreboard:updated { entries }
```
Client applies events optimistically and reconciles against REST on reconnect.

## Screens
1. **Auth** — register, login
2. **Onboarding** — create a household, or land here from an invite link
3. **Board** — the main surface. Unclaimed tasks in a shared column; each member's claimed tasks beside it. Add task via a floating button. Tap to open detail.
4. **Task detail** — edit, claim, complete, attach photo
5. **Scoreboard** — current week, both members side by side, plus last week's result
6. **Rewards** — list, create, redeem
7. **Settings** — profile, household members, invite link, sign out

## Build order
Ship each phase working before starting the next.

1. **Foundations** — repo skeleton, DB, migrations, auth endpoints, JWT middleware, register/login screens
2. **Households** — create, invite link generation, accept flow, member list, onboarding screens
3. **Tasks** — full CRUD over REST, Board and Task Detail screens, claim/complete
4. **Realtime** — Socket.IO layer, live board updates on both devices
5. **Scoreboard** — score periods, weekly rollover, scoreboard screen
6. **Rewards** — rewards CRUD, redemption, points balance
7. **Photo proof** — upload to bucket, thumbnail on task detail

## Non-goals for v1
No calendar sync, no recurring tasks, no households larger than 6, no web client, no push notifications (phase 8 candidate), no "punishments" — rewards only until the core loop proves out.
