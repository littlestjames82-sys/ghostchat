# GhostChat — Rebrand Notes

GhostChat is a local rebrand of the open-source **Chatwoot** customer-messaging
platform for Ghost Developer Studio. Nothing here is deployed or public.

- Upstream: https://github.com/chatwoot/chatwoot (version 4.18.0)
- Cloned at commit: `843385ff3a01c9c0d35646bb45a0e5e9950477b2` (2026-10-01)
- License: MIT for everything outside `enterprise/` — `LICENSE` untouched.
  `enterprise/` is under its separate upstream license and was **not** modified.
- Attribution: "Based on Chatwoot (MIT)" in the README header.
- Ryan's pick, Oct 3, 2026: item #3 (Chatwoot) from the open-source shortlist,
  "but rebrand it" — same treatment as GhostCut (OpenCut → GhostCut).

## How the rebrand was done

Chatwoot is built to be white-labeled: the dashboard, login, widget and emails
all read the installation brand from `config/installation_config.yml` (seeded
into the DB on install) and `public/brand-assets/`. The rebrand works through
those sanctioned surfaces first, plus the theme palette and static icons, so
upstream updates stay easy to pull.

### Product identity (user-visible)
- `config/installation_config.yml` (branding section only):
  `INSTALLATION_NAME` and `BRAND_NAME` → `GhostChat`; `LOGO` / `LOGO_DARK` /
  `LOGO_THUMBNAIL` now point at the new GhostChat PNGs in `public/brand-assets/`;
  `BRAND_URL` / `WIDGET_BRAND_URL` → the studio Facebook page
  (https://www.facebook.com/ghostdeveloperstudio). `TERMS_URL` / `PRIVACY_URL`
  also point at the studio page for now — **replace with real terms/privacy
  pages before any public use.**
- `public/manifest.json` — PWA name/short_name `GhostChat`, colors → violet.
- `app/views/installation/onboarding/index.html.erb` — title `SuperAdmin | GhostChat`.
- `package.json` — name `@chatwoot/chatwoot` → `@ghostchat/ghostchat`.
- `.env.example` — `MAILER_SENDER_EMAIL=GhostChat <gdev6145@gmail.com>`.
- `README.md` — new GhostChat header + attribution on top; upstream README kept below.

### Logo & icons
- New wide wordmarks rendered from the studio code-ghost logo
  (violet ghost on charcoal, `ghost-dev-studio/logos/ghost-developer-studio-logo.png`):
  `public/brand-assets/logo.png` (dark text, light mode) and `logo_dark.png`
  (white text, dark mode), 1760×400. `logo_thumbnail.png` is the 512×512 tile.
  The upstream SVG wordmarks are still in that folder but no longer referenced
  by the community config.
- All 24 non-badge PNG icon slots in `public/` (android/apple/favicon/ms sizes)
  regenerated from the studio logo at their exact original dimensions.
  Skipped on purpose: `favicon-badge-*` (notification-badge variants, not the
  app icon) and the two `apple-touch-icon*.png` files, which are 0-byte
  placeholders upstream.

### Accent color: Chatwoot blue → studio violet
- `theme/colors.js` — the `woot` palette now maps to the Radix **violet /
  violetDark** scale instead of blue/blueDark (same slot mapping, hue swap,
  same approach as GhostCut), and `n.brand` `#2781F6` → `#6716F3`.
- Every remaining `#2781F6` in user-visible app code was swapped to `#6716F3`:
  dashboard Logo component, portal dialog, message/widget components, devise
  confirmation mailer, mailer base layout, `vueapp.html.erb` theme-color/TileColor,
  `theme/icons.js`, `public/manifest.json`. Verified: **zero** `#2781F6` left
  outside `enterprise/`.
- Left alone on purpose: `#1F93FF` — that's the default color for *new
  user-created* widgets/portals/labels (content defaults in models/DB), not
  the installation brand.

## What was intentionally left alone
- `LICENSE`, upstream copyright notices, and the whole `enterprise/` tree.
- Internal identifiers: `window.chatwootConfig`, `ChatwootApp`, the
  `chatwoot` Rails/DB naming, widget SDK protocol, and env var names
  (`CHATWOOT_*` installation configs). Renaming those would break the
  widget/API protocol and upstream compatibility — same call as leaving
  `opencut-wasm` alone in GhostCut. They are not user-visible brand surfaces.
- `enterprise/config/premium_installation_config.yml` still points LOGO at the
  old SVGs — only relevant on a premium/enterprise plan, which this isn't.

## Verification (Oct 3, 2026)
- `config/installation_config.yml` parses as valid YAML; all 9 branding
  values read back as GhostChat / studio URLs / new logo paths.
- `public/manifest.json` and `package.json` parse as valid JSON with the new
  name/colors.
- `node --check theme/colors.js` passes (same result as the pristine file).
- All new/regenerated PNGs open and are at the correct dimensions.
- `git diff`: 44 files changed, all in the list above.

## NOT verified — needs a real stack
- **The app was not booted.** Chatwoot is a Rails + Vue app that needs
  Ruby 3.4.4 (`.ruby-version`), PostgreSQL, Redis and Sidekiq; this sandbox
  has no Ruby, no Docker and no pnpm, so no install/build/run was possible
  here. First real run should use the repo's `docker-compose.yaml` on Ryan's
  machine/VM, then confirm the GhostChat name, logo and violet palette on the
  login page and dashboard, and the widget on a test page.
- Emails, the help-center portal branding, and push notifications inherit
  the same installation config but were not exercised.
- Nothing deployed, nothing public, no APK (this is a server app; a phone
  APK shell like DevPulse/GhostCut's only makes sense once it's hosted
  somewhere reachable).

## GitHub (Oct 3, 2026)
- Pushed at Ryan's request to **https://github.com/littlestjames82-sys/ghostchat**
  (public fork of chatwoot/chatwoot, named `ghostchat` at fork time; repo
  description set to GhostChat / Ghost Developer Studio).
- The sandbox has no git-push credential, so the rebrand went up through the
  GitHub Git Data API: blobs only for the 45 changed/new files on top of the
  fork's upstream tree (commit `843385f`), then a tree + commit + ref update
  on `develop`.
- Commits: `7c5678b` (rebrand), `d4913f5` (restores upstream 100755 mode on
  ChatInputWrap.vue).
- Verified after push: remote recursive tree = 9,509 blobs, **0 differences**
  (path, mode, SHA) against the local rebranded tree.
