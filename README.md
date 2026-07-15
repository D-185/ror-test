# Todo App

A simple Rails 8 Todo list app, backed by PostgreSQL, generated as a standard/production-ready starting point.

## Stack

* Ruby 3.3.11, Rails 8.1.3
* PostgreSQL 17 (adapter: `postgresql`)
* Tailwind CSS, Turbo, Stimulus (Hotwire, no Node build step needed via importmaps)
* Solid Cache / Solid Queue / Solid Cable (DB-backed, no Redis needed)
* Kamal (Docker-based deployment), Brakeman + bundler-audit (security scanning), RuboCop (style)

## Local setup

1. Copy `.env.example` to `.env` and fill in your local PostgreSQL credentials (already done for this machine — see `.env`, which is git-ignored).
2. Install gems:
   ```
   bundle install
   ```
3. Create and migrate the databases:
   ```
   bin/rails db:prepare
   ```
4. Start the server:
   ```
   bin/rails server -p 3000
   ```
5. Visit http://localhost:3000 — the Todos list is the app's root page.

## Database

* Dedicated PostgreSQL role `todo_app` (not the `postgres` superuser) owns this app's data — least-privilege, matches how production should be configured.
* Connection settings come from environment variables (`DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_USERNAME`, `DATABASE_PASSWORD`), loaded from `.env` in development via the `dotenv-rails` gem. See `config/database.yml`.
* In production, set `TODO_APP_DATABASE_PASSWORD` (and the `DATABASE_*` vars) as real environment variables / secrets — never commit `.env`.

## Common commands

| Command | Purpose |
|---|---|
| `bin/rails server` | Start dev server |
| `bin/rails console` | Open REPL |
| `bin/rails generate scaffold Name field:type` | Scaffold a new resource |
| `bin/rails db:migrate` | Run pending migrations |
| `bin/rails test` | Run the test suite |
| `bin/rubocop` | Lint (Omakase style) |
| `bin/brakeman` | Security static analysis |
| `bundle exec kamal deploy` | Deploy via Docker (after configuring `config/deploy.yml`) |

## Todo resource

Generated via `rails generate scaffold Todo title:string description:text completed:boolean`. Full CRUD is available at `/todos`.
