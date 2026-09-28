# Household Task Board — Build Spec v1

## Product summary

A mobile app for people living together (target: younger couples) to fairly split household duties. The core surface is a shared board — a lightweight whiteboard where either person can post a task, assign it, or claim it. Completed tasks earn points; points feed a weekly scoreboard and unlock rewards the household defines itself.

Tone: playful, joint effort. Explicitly **not** a surveillance or nagging tool. Both members see the same board and the same numbers.

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

> **TODO:** This section has not been written yet. Do not invent tables until it is filled in.
