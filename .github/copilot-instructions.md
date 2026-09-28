# Copilot instructions

The product and build spec live in [`docs/SPEC.md`](../docs/SPEC.md). Read it before generating code.

Follow these conventions in every change:

- JavaScript across the stack: Expo (React Native) app, Node + Express backend, Postgres, Socket.IO.
- One module per concern; no multi-purpose files.
- One file per model, one file per route group, one file per API resource on the client.
- Named exports only; default exports are allowed for React components only.
- All DB access goes through model modules — routes never write SQL.
- Migrations are numbered and forward-only; never edit an existing migration.
- Validate input at the route boundary before touching a model.
- Secrets go in `.env`; only `.env.example` is committed.
- Keep the README current: setup, migrations, and how to run the app and the server.
- Tone of user-facing copy: playful and cooperative, never nagging or surveillance-like.
