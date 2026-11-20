# Repository Guidelines

## Project Overview

Buzz is a self-hostable workspace for humans and agents built around a Nostr relay. A community is selected by its URL and is the tenant isolation boundary. Read `VISION.md`, the relevant `VISION_*.md`, and applicable `TESTING.md` guides before non-trivial changes; implementation is not necessarily the product direction.

## Architecture & Data Flow

- Clients and agents send signed Nostr events to the relay over WebSocket. The relay resolves the community from the host before authentication or data access; unknown hosts fail closed. It verifies and authorizes events, persists them in Postgres, and fans them out locally and through Redis. Search uses Postgres; audit and workflow subsystems handle their respective side effects. See `ARCHITECTURE.md`.
- `buzz-relay` composes focused services through `AppState`; cross-subsystem coordination belongs in the relay. Its handlers and subscription registry manage event ingestion and delivery. Tokio cancellation and bounded channels coordinate connection lifetimes and backpressure.
- Prefer Nostr events to new feature-specific HTTP APIs. Define event kinds in `crates/buzz-core/src/kind.rs`; agent-facing operations belong in `buzz-cli`. HTTP also serves bridge queries, media, git, webhooks, health, and other appropriate backend surfaces. Protected HTTP routes must follow the NIP-FI admission rules in `crates/buzz-relay/src/router.rs`.
- Channel events use `h` tags, not `e` tags; channel metadata and membership addressable events use `d` tags. Relay queries need explicit `kinds`; preserve channel membership and community isolation on every read and write.
- Desktop communicates with Tauri-managed Rust state through IPC; React providers and TanStack Query manage client state. Mobile uses Riverpod providers and hooks; its intended offline-first store is derived from verified events, with drafts and pending actions stored separately. The web repo browser uses React, TanStack Query, and Nostr WebSockets.

## Key Directories

- `crates/` — Rust workspace: `buzz-relay`, `buzz-core`, `buzz-db`, `buzz-auth`, `buzz-pubsub`, `buzz-search`, `buzz-audit`, `buzz-workflow`, `buzz-cli`, `buzz-acp`, and supporting crates.
- `desktop/` — React desktop frontend and separate Tauri Rust workspace in `src-tauri/`; features live under `src/features/`.
- `mobile/` — Flutter app, feature modules in `lib/features/` and shared infrastructure in `lib/shared/`.
- `web/`, `admin-web/` — browser clients in the pnpm workspace.
- `migrations/`, `schema/` — database migrations and desired-state schema; `scripts/` contains development and verification tooling.
- `crates/buzz-test-client/tests/`, `desktop/tests/e2e/`, `mobile/test/` — integration and client test suites.

## Development Commands

Activate Hermit in each shell, including before Git or hooks:

```bash
. ./bin/activate-hermit
just setup              # bootstrap .env, services, migrations, dependencies
just dev                # relay and desktop; or just relay / just desktop-dev
just build              # Rust workspace
just check              # lint, formatting, and policy checks
just test-unit          # infrastructure-free unit lane
just test               # unit and integration tests; starts Docker if needed
just ci                 # pre-PR checks, unit tests, and client builds
```

Use `just desktop-check`, `just desktop-typecheck`, `just desktop-test`, `just web-check`, or `just mobile-check` / `just mobile-test` for focused work. Tauri is excluded from the root Cargo workspace: use `just desktop-tauri-fmt-check`, `just desktop-tauri-clippy`, and `just desktop-tauri-test` separately. `just mobile-dev` runs the app on an iOS simulator with worktree-specific debug identity. `just reset` deletes development data and can affect a desktop instance sharing its services; do not run it casually.

## Code Conventions & Common Patterns

- Match the existing feature and package conventions. Rust uses `rustfmt`, Clippy, typed `thiserror` errors for libraries, `anyhow` for binaries, and structured `tracing`. No new `unsafe`, or `unwrap()` / `expect()` in production; document public APIs.
- Propagate failures or retain durable retry state; do not turn failures into success. Fence stale async results, bound resources and retry loops, and persist a user action atomically or make recovery durable. Preserve recovery affordances and overlap history backfill with live subscriptions.
- Desktop readable text uses named rem-based Tailwind tokens, including `text-message`, `text-2xs`, and `text-3xs`; do not add arbitrary px/rem text sizes. Community switches remount React state but not module singletons: register community-scoped resets in `desktop/src/features/communities/useCommunityInit.ts`.
- Mobile uses `ConsumerWidget` / `HookConsumerWidget`, Riverpod and hooks, not `StatefulWidget`. Feature modules import shared code rather than other features. Use `Grid`, `Radii`, `context.colors`, and `context.textTheme`; keep widgets small and files under the enforced 1200-line ceiling.
- Include accessible semantics and keyboard, pointer, and other applicable input paths for interactive UI. For mention editing, preserve exact identity labels, caret behavior, and visible ambiguity errors; see `docs/mention-editor.md`.
- For migrations, schema, or schema-dependent `buzz-db` queries, read `docs/database.md`. Never edit merged migrations or add triggers, functions, foreign keys, or migration backfills; update all applicable schema locations and maintain compatibility with the deployed relay. Follow its expand/deploy/backfill/contract rules and lock timeout requirements.

## Important Files

- `crates/buzz-relay/src/main.rs`, `router.rs`, `state.rs`, `tenant.rs`, `handlers/` — relay composition, routing, tenant binding, and event handling.
- `crates/buzz-core/src/kind.rs`, `crates/buzz-cli/src/main.rs`, `crates/buzz-acp/src/main.rs` — protocol registry and agent entry points.
- `desktop/src/main.tsx`, `desktop/src-tauri/src/lib.rs`, `mobile/lib/main.dart`, `web/src/main.tsx` — client entry points.
- `Justfile`, `Cargo.toml`, `rust-toolchain.toml`, `pnpm-workspace.yaml`, `mobile/pubspec.yaml`, `.env.example` — commands, toolchains, dependencies, and configuration.
- `CONTRIBUTING.md`, `TESTING.md`, `docs/database.md`, `RELEASING.md` — contributor, testing, schema, and release details.

## Runtime/Tooling Preferences

- Use the Hermit-pinned Rust, Node, pnpm, Flutter, and `just` tools; see `rust-toolchain.toml` and `bin/` for versions. The JavaScript workspace uses pnpm, not Bun; install from the root with `pnpm install`. Docker supplies local Postgres and Redis. Linux Tauri builds also need system GTK/WebKit libraries.
- `just setup` installs hooks; pre-commit hooks can modify and restage formatting changes. Commits require a DCO sign-off (`git commit -s`). Worktree UI ports and mobile debug identities are isolated, but default relay services may be shared.

## Testing & QA

- Use the narrowest relevant test during development and `just ci` before a PR; Flutter/Dart-only changes may use `just mobile-install mobile-check mobile-test` instead. Relay, database, and auth changes also require `just test`. `just ci` does not include the dedicated PostgreSQL lane or browser E2E suites.
- Rust uses Cargo/nextest; desktop unit tests use Node's test runner and jsdom; desktop and web E2E use Playwright; mobile uses Flutter tests. Desktop mock smoke: `cd desktop && pnpm test:e2e:smoke`; relay-backed: `pnpm test:e2e:integration`. Both build the E2E bridge; a normal desktop build does not. Register new desktop specs in `desktop/playwright.config.ts`.
- Ignored relay E2E tests require a running relay: `cargo test -p buzz-test-client -- --ignored`. For PostgreSQL tests use `scripts/postgres-test-run.sh` and the isolation/naming rules in `crates/buzz-db/TESTING.md`; never hard-code the shared development database.
- Regression tests must exercise the production seam and fail if the guard is removed. Prefer behavioral journeys and test failures, accessibility, and non-primary input where applicable. Exercise changed user-visible or integration behavior in the actual app, relay, or CLI when practical; mock browser evidence is not native-app evidence.
- Open work-in-progress PRs as drafts. Before marking ready, obtain agent review, exercise the change, and give the human exact test steps; only the human can confirm their own test. Add `buzz-review-completed` only after those steps, and remove it if later behavioral edits require retesting. UI PRs need before/after screenshots or recordings; use `scripts/post-screenshots.sh`, not relay media URLs. See `CONTRIBUTING.md` and `TESTING.md` for full workflows.
