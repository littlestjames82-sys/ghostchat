# GhostChat Boot Proof — 2026-10-05

## Status: ✅ BOOTED in the sandbox — the full stack installed natively and served the GhostChat-branded app

Ryan approved installing the stack ("Can you install those 4" — Docker, Ruby,
Postgres, Redis). The native route succeeded: GhostChat (Chatwoot 4.18 rebrand)
is running locally with real Postgres + Redis behind it, and the sign-in page
renders the GhostChat brand — ghost logo, "Login to GhostChat", studio-violet
button — from the seeded installation config.

**The stack is LEFT RUNNING** (all listeners on 127.0.0.1 only):
Rails/Puma :3100 · Vite dev server :3036 · Sidekiq (health :7433) ·
Postgres :5432 · Redis :6379. Control: `~/workspace/tools/ghostchat-boot/stack.sh start|stop|status`.
Local super admin: `admin@ghostdev.local` — generated password is in
`~/workspace/tools/ghostchat-boot/admin-credentials.txt` (chmod 600, not in the repo).

## What was installed, and by which route

| Piece | Version | Route |
|---|---|---|
| Ruby | 3.4.4 (+PRISM) | rbenv + ruby-build, compiled from source. System libyaml headers were missing, so libyaml 0.2.5 was compiled from the official tarball into /usr/local first. |
| Bundler / gems | Bundler 2.5.16 · 382 gems | `bundle install` against the repo Gemfile.lock; `pg` gem compiled against the bundled Postgres' `pg_config`. `bundle check` passes. |
| Postgres | 16.2 | The pip package **`pgserver`** (venv at `~/workspace/tools/pgvenv`) — it bundles real Postgres binaries. Mirror copy kept at `~/workspace/tools/pginstall` (+ `pginstall-libs`). |
| Postgres extensions | pgvector (bundled) + pg_stat_statements, pg_trgm, pgcrypto | Schema needs all four. pgvector ships with pgserver; the other three were compiled from the official Postgres 16.2 source via PGXS against the bundled install. pgcrypto needed an explicit `-lcrypto` link (the bundled server binary is not linked to OpenSSL). `shared_preload_libraries='pg_stat_statements'` set; all four `CREATE EXTENSION` verified. |
| Redis | 8.10.2 | Compiled from the official `redis-stable` source tarball. Core server built fine; the optional search/json/timeseries modules failed to build and are not needed by Chatwoot. |
| Node deps | pnpm 10.34.6 · 1,073 packages | `pnpm install` (6m 36s). The repo pins `pnpm@10.2.0` via `packageManager`; pnpm's self-management download failed in this sandbox, so `manage-package-manager-versions=false` is set in `~/.config/pnpm/rc` and the installed 10.34.6 (satisfies the `10.x` engine) is used. |
| Docker | docker.io 29.1.3 (client + daemon binaries) | apt cooperated on one bounded run and the package installed — but **dockerd cannot start here**: network-controller init fails on iptables NAT rules (missing kernel module / userns limits). Docker is therefore *installed but unusable in this sandbox*; it was not needed for the boot. The deploy kit's Docker route still belongs on a Docker-capable host. |

## Sandbox traps found (why the first attempts died)

- **`/tmp` is a 512MB tmpfs.** The first Ruby compile and the first Redis link
  both died with `No space left on device`, and a previous agent's build died
  the same way. Fix: all build trees, caches and TMPDIR were moved under
  `~/workspace/tools/ghostchat-boot/` (63GB free). Evidence: `ruby-build.log`,
  `redis-build.log` (first attempts) vs `ruby-build2.log`, `redis-build2.log`.
- **The workspace filesystem forbids `chown`** (userns), and Postgres refuses to
  run as root — so Postgres runs as user `nobody` with its binaries and data
  dir in `/tmp` (`/tmp/pgsrv`, `/tmp/pgdata`). Consequence: **the Postgres data
  directory does not survive a VM replacement**; the durable copies of the
  binaries are in the workspace and the DB can be rebuilt with the commands
  below.
- **The apt mirror is unreliable** (long stalls, failed index fetches). All
  successful downloads came via `curl` through the egress proxy.
- **Raw DNS (UDP 53) is blocked** for sandbox processes. This is why the web
  onboarding form could not create the first admin: Chatwoot validates the
  email domain with a DNS/MX lookup, which raises `Errno::EPERM` here. The
  super admin was therefore seeded with `rails runner` (identical records —
  Account, SuperAdmin user, administrator AccountUser — with the email-format
  validation skipped), and sign-in was then proven through the real auth API.
  On any normal host (Ryan's machine, a VPS) onboarding works unmodified.
- **RAM: 2 cores / ~8GB, shared with a concurrent GhostCut builder.** The
  production Vite build of the main dashboard bundle (5,139 modules) was
  OOM-killed twice during chunk rendering, even with a 3GB Node heap. The
  widget/sdk bundles built fine (`public/packs`, 43MB). The app is therefore
  served via the repo's own development route (`bin/vite dev` + Rails +
  Sidekiq, per `Procfile.dev`) — on-demand module serving instead of the
  production bundle. A production precompile should be re-run when the box
  isn't shared, or on the deploy host.

## Boot evidence

- Postgres: `SELECT version()` → PostgreSQL 16.2; `chatwoot_production` +
  `chatwoot_test` created; after `db:chatwoot_prepare` (exit 0): **108 tables**.
- Redis: `PING` → `PONG`; Sidekiq booted against it and its cron jobs are cycling.
- Installation config in DB: `INSTALLATION_NAME = GhostChat`,
  `BRAND_NAME = GhostChat`, `LOGO = /brand-assets/logo.png`,
  `BRAND_URL = https://www.facebook.com/ghostdeveloperstudio`.
- Rails: `GET /` → 302 → `/installation/onboarding` (HTTP 200,
  `<title>SuperAdmin | GhostChat</title>`); after seeding the admin,
  `GET /app/login` → HTTP 200.
- Auth: `POST /auth/sign_in` with the seeded admin returned a session
  (access token, `account_id: 1`, `role: administrator`, `type: SuperAdmin`).
- Screenshots (in `~/workspace/tools/ghostchat-boot/`):
  - `ghostchat-login.png` — **"Login to GhostChat" with the ghost logo and a
    violet Login button**: the runtime rebrand, proven.
  - `ghostchat-onboarding.png` — the first-run setup form. Honest caveat: this
    one view still shows the upstream wordmark — its template references the
    leftover `/brand-assets/logo.svg` upstream files and a hardcoded welcome
    string, which the white-label config does not drive. The page title,
    the seeded config, the login page, and the violet theme are all GhostChat.
    Fixing that template is a small rebrand follow-up, not done here.
  - `boot-evidence.html` / `login-evidence.html` — raw served HTML.
- Logs for every step are in `~/workspace/tools/ghostchat-boot/`
  (`ruby-build2.log`, `bundle-install.log`, `pnpm-install3.log`,
  `db-prepare2.log`, `rails-server.log`, `sidekiq.log`, `vite-dev.log`,
  `dockerd.log`, `docker-apt.log`, `contrib-build.log`).

## Start / stop

```bash
~/workspace/tools/ghostchat-boot/stack.sh start   # postgres, redis, vite, sidekiq, rails
~/workspace/tools/ghostchat-boot/stack.sh status  # rails HTTP code, redis PING, psql SELECT 1
~/workspace/tools/ghostchat-boot/stack.sh stop
# App: http://127.0.0.1:3100  (sandbox-local only, not exposed)
```

Rebuilding the database from scratch (e.g. after a VM replacement wipes /tmp):

```bash
source ~/workspace/tools/ghostchat-boot/env.sh
~/workspace/tools/ghostchat-boot/pg-start.sh start          # initdb first if /tmp/pgdata is gone:
#   su -s /bin/bash nobody -c "LD_LIBRARY_PATH=/tmp/pgsrv/pgserver.libs:/tmp/pgsrv/pgserver/pginstall/lib /tmp/pgsrv/pgserver/pginstall/bin/initdb -D /tmp/pgdata -U postgres --auth=trust"
cd ~/workspace/ghostchat && bundle exec rails db:chatwoot_prepare
```

## Honest scope

- This is a **boot proof on a shared sandbox**: development-mode serving,
  localhost only, no mail delivery, no production hardening. It proves the
  rebranded application installs, migrates, seeds, authenticates, and serves
  its branded UI on a real Postgres + Redis — the thing that was unproven.
- The production asset precompile of the main dashboard bundle remains
  unproven *in this sandbox* (RAM, above); the Docker production path remains
  the deploy kit's route on a Docker-capable host.
- `.env` (repo root, chmod 600, git-ignored) holds locally generated secrets;
  none are committed or reproduced here.
- **Captain AI: premium-gated in upstream Chatwoot 4.18 — untouched, no bypass
  attempted.** The legitimate path is Chatwoot's paid plan when revenue-ready.
- TERMS_URL / PRIVACY_URL in the installation config still point at the studio
  Facebook page (pre-existing rebrand placeholder) — replace before any
  public use, as already noted in REBRAND_NOTES.md.

---

## Addendum — 2026-10-05 (later): restart wipe + restore, stack now self-healing

A sandbox restart wiped `/tmp`, taking the Postgres binaries and data
(`/tmp/pgsrv`, `/tmp/pgdata`) with it — exactly the failure mode flagged
above. Redis (workspace `dl/`), Ruby, gems and node modules survived. The
stack has been **fully restored and re-proven**:

- **Postgres restored from the workspace mirror** — no rebuild from source
  was needed: `~/workspace/tools/pginstall` already held the complete
  install (binaries + all four extension `.so`/SQL files: pgvector,
  pg_stat_statements, pg_trgm, pgcrypto) and `pgserver.libs` survived in
  the pgvenv site-packages. Copied back to `/tmp/pgsrv`, fresh `initdb`
  (`pg-initdb-restore.log`), `shared_preload_libraries='pg_stat_statements'`
  re-applied, server started: PostgreSQL 16.2.
- **DB rebuilt**: `rails db:chatwoot_prepare` exit 0 → `chatwoot_production`,
  108 tables, all four extensions present (`db-prepare-restore.log`). The
  `.env` expects the `postgres` superuser, so no extra role was needed.
- **Seed restored** (`seed-restore.rb`, idempotent): installation config
  back to `INSTALLATION_NAME = GhostChat`, `BRAND_NAME = GhostChat`,
  `BRAND_URL`/`TERMS_URL`/`PRIVACY_URL` = the studio Facebook page, and the
  **same super admin recreated** (`admin@ghostdev.local`, account_id 1,
  administrator AccountUser; password unchanged, still only in
  `admin-credentials.txt`, chmod 600).
- **Re-proven end-to-end**: `GET /` → HTTP 200; `GET /app/login` → HTTP 200
  with `"INSTALLATION_NAME":"GhostChat"` / `"BRAND_NAME":"GhostChat"` in the
  served page; `POST /auth/sign_in` → HTTP 200 (`account_id: 1`,
  `role: administrator`, `type: SuperAdmin`); `stack.sh status` → rails 200,
  Redis PONG, psql `SELECT 1`; Puma, Sidekiq and the Vite dev server all up.

**New traps found during the restore** (all fixed in the scripts):

- The workspace mirror copy needs `chmod -R a+rX` after being copied to
  `/tmp`, or user `nobody` gets "Permission denied" executing the binaries.
- `db/seeds.rb` sets the Redis flag `CHATWOOT_INSTALLATION_ONBOARDING`,
  which parks every page on `/installation/onboarding`; the restore seed
  now deletes it (mirroring a completed onboarding). A stale copy also
  **resurrected from Redis' RDB snapshot** after a plain `DEL` — fixed with
  `DEL` + `BGSAVE`.
- `bin/vite dev` died with `pnpm: No such file or directory` — the pnpm
  bin dir (`~/workspace/tools/pnpm/node_modules/.bin`) is now on the PATH
  in `env.sh`.
- Ordering: seeds touch Redis (`GlobalConfig.clear_cache`), so Redis must
  be started **before** any DB prepare. `stack.sh` now does this.

**Self-healing start (one command after any future sandbox restart):**

```bash
~/workspace/tools/ghostchat-boot/stack.sh start
```

`pg-start.sh start` now runs an `ensure` step first: missing binaries are
re-copied from the workspace mirror, a missing data dir is re-`initdb`'d
and marked with `/tmp/pgdata/.needs_prepare`; `stack.sh` sees the marker,
starts Redis, re-runs `db:chatwoot_prepare` + `seed-restore.rb`, clears
the marker, then boots Vite/Sidekiq/Rails. Verified by simulating a full
wipe (`rm -rf /tmp/pgsrv /tmp/pgdata`) and recovering with that single
command — log line: `[stack] DB rebuilt and re-seeded`, followed by the
HTTP 200 + auth proof above. Explicit route if ever needed:
`stack.sh rebuild-pg` (stop, wipe /tmp pg state, re-ensure).

**Status: ✅ RESTORED and LEFT RUNNING** on the same 127.0.0.1 listeners
(Rails :3100 · Vite :3036 · Sidekiq · Postgres :5432 · Redis :6379).
