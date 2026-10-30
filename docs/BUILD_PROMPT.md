# PitWall — Master Build Prompt

> Paste everything below the line into a fresh agent session running in this repository.
> To resume a partial build, append: *"Resume from Phase N, task N.M — audit completed work first, then continue."*

---

You are the lead engineer building **PitWall**, a real-time motorsport statistics hub. Your single source of truth is **`docs/BLUEPRINT.md` (v1.1)** in this repository — read it in full before writing any code. This prompt governs *how* you execute it. If this prompt and the blueprint conflict, the blueprint wins. Where both are silent, make the pragmatic choice and record it in `docs/DECISIONS.md` (one line per decision: date, choice, why).

## 1. Mission

Build PitWall v1 end to end — blueprint Phases 1–4 — as a production-ready Django monolith deployable to a single VPS via Docker Compose. **Formula 1 only at launch**; the database schema stays multi-series. Work autonomously: plan, build, verify, commit, repeat.

## 2. Locked decisions — never revisit, never substitute

| Area | Decision |
|---|---|
| Scope | F1 only; other 8 series are schema-ready but have no ingest or pages |
| Backend | Django 5, Python 3.12, ASGI via Uvicorn |
| Frontend | Django templates + Bootstrap 5.3 + HTMX + Alpine.js — all vendored static files, **no Node/npm anywhere** |
| Real-time | Server-Sent Events (async views + Redis pub/sub) — **no Django Channels, no WebSockets** |
| Auth | django-allauth, session-based: email/password + Google only — **no JWT, no Apple/Facebook** |
| API | **No DRF / public JSON API in v1** (templates read the ORM; JSON only for push-subscribe and card-generate endpoints) |
| Admin | Django built-in admin + custom staff views only where it falls short |
| Data | OpenF1 (live) + Jolpica (canonical/historical); backfill seasons 2020 → current |
| Tasks | Celery + Celery Beat (django-celery-beat), Redis broker |
| Store | PostgreSQL 16, Redis 7 |
| Ops | Docker Compose (dev + prod), Nginx in front, Dokploy-compatible |
| Charts | Chart.js (vendored) |
| Images | Pillow only (FFmpeg for Phase 3 video) |

## 3. Working rules

- Execute **phase by phase, task by task**, in blueprint §14 order. Track progress with a todo list; one task in progress at a time.
- **Functional patterns preferred**: thin views, pure service functions (`apps/*/services.py`), explicit data in/out. Business logic never lives in views or templates. Use classes only where Django requires them.
- **Commits**: one per completed task, short imperative message (e.g. `add f1 standings page`). The app must run, lint, and pass tests before every push.
- **Never commit secrets.** `.env.example` must always list every variable the code reads, with safe defaults for dev.
- Quality bar: type hints on all public functions, docstrings on services, `ruff format` + `ruff check` clean, `pytest` green.
- Keep dependencies minimal and pinned (`requirements/base.txt`, `requirements/production.txt`).
- Do not scrape or embed copyrighted assets: no official F1/team logos, no driver photos. Use text, team colours, and generated initials-avatars/placeholder SVGs; keep `*_url` fields for later. Self-host fonts (Inter + JetBrains Mono) as static files — no third-party CDNs.

## 4. Engineering requirements — the cover-all-bases checklist

### Data & ingestion
- HTTP clients (`apps/ingest/clients/`): httpx, 10s timeouts, retries with exponential backoff + jitter, honour `Retry-After` on 429.
- Respect Jolpica rate limits (≤4 req/s burst, ≤500 req/h): throttle the backfill, batch politely.
- **Idempotent everywhere**: all writes use `update_or_create` keyed on natural unique constraints; re-running any ingest task or the whole backfill is always safe.
- `python manage.py backfill_f1 --from-season 2020` — seasons, rounds, sessions, drivers, teams, entries, results, qualifying, standings.
- `python manage.py seed_series` — creates the 9 series rows (only F1 active).
- Every ingest run recorded in an `IngestLog` model (source, task, started/finished, status, message, items written) — drives the admin health dashboard.
- Dispatcher pattern per blueprint §3.4: beat runs `tick` every minute; a Redis flag gates a self-rescheduling 5s fast-poll loop. Guarantee no overlap (lock) and no orphan loops (flag TTL).
- Reconciliation: OpenF1 data is provisional; Jolpica canonical results overwrite after each round.

### Live layer
- SSE exactly per blueprint §7: async view, Redis pub/sub, named events, snapshot key for late joiners, 15s heartbeat comment, `X-Accel-Buffering: no`.
- **Build a replay simulator with the live layer, not after**: `python manage.py simulate_live <session_id> [--speed 10]` replays synthetic (or recorded OpenF1) frames through the *real* pipeline (Redis publish → SSE → UI), so live timing is fully demoable and testable off-season. This is the acceptance vehicle for all live features.

### Correctness edge cases (all must be handled and tested)
- Sprint weekends: all six session types; sprint vs race points split.
- Mid-season team changes: `driver_season_entries` is per-season; `race_results` snapshots the team at race time.
- Result statuses: Finished, +N laps, DNF, DNS, DSQ; shared/no fastest lap; half points; points-paying fastest lap rules by season.
- Cancelled/postponed sessions and rounds.
- Off-season and between-rounds: home/hub pages render gracefully with "no upcoming round"; countdown handles absence.
- Source downtime: serve stale DB data with a visible "last updated" timestamp; an API being down must never 500 a page.
- Timezones: store UTC everywhere; render in the viewer's local timezone client-side with a tz label.

### Frontend
- Bootstrap 5.3 vendored (`bootstrap.min.css` + `bootstrap.bundle.min.js`); brand/series theming via CSS-variable overrides in `static/css/pitwall.css`; series accent colours from blueprint Appendix A.
- Dark mode: `data-bs-theme` on `<html>`; default follows `prefers-color-scheme`; Alpine toggle persists to localStorage and to the user profile when logged in; no flash-of-wrong-theme (inline head script).
- HTMX + django-htmx middleware; CSRF wired for htmx POSTs; fragments live in `templates/partials/`.
- Responsive at 375/768/1280; timing tables scroll horizontally on mobile.
- Accessibility WCAG 2.1 AA: semantic tables with header scopes, visible focus states, AA contrast in both themes, `aria-live="polite"` on live-updating regions, skip-to-content link.
- SEO: `django.contrib.sitemaps`, canonical URLs, per-page title/meta/OG tags, `robots.txt`.

### Auth & accounts
- **Custom User model in the very first migration** (never retrofit), per blueprint §5 users table.
- allauth: email/password with mandatory verification (console email backend in dev) + Google OAuth. Styled allauth templates matching the site shell.
- Session cookies httpOnly, Secure in prod, SameSite=Lax; rate-limit login/signup/password-reset.
- GDPR: cookie-consent banner; self-service account deletion cascading all personal data; privacy page stub.

### Notifications (Phase 2)
- Preference matrix (channel × event type × entity) enforced in a single dispatch service; every send logged to `notification_log`.
- Web Push via pywebpush + VAPID (document key generation in README); prune subscriptions on 410; service worker `static/js/sw.js` handles push + notificationclick.
- Emails: HTML + plain-text alternatives for every template.

### Security & settings
- Settings split base/development/production; prod: `DEBUG=False`, `SECURE_*` headers, HSTS, proxy SSL header, allowed hosts from env.
- `/healthz` endpoint checking DB + Redis (used by Docker healthchecks).
- Structured logging; request IDs; Celery task logging to the same scheme.

### Testing & CI
- pytest + pytest-django + factory_boy; mock HTTP with respx. Cover: ingest parsers/clients, standings math, points edge cases, dispatcher gating, SSE frame formatting, preference dispatch logic, and smoke tests (200 + key content) for every public page.
- GitHub Actions: ruff + pytest on push/PR (services: postgres, redis).
- `Makefile`: `setup`, `dev`, `test`, `lint`, `backfill`, `simulate` targets.

### Ops
- `docker-compose.yml` (dev: bind-mount, autoreload, console email) and `docker-compose.prod.yml` per blueprint §13.
- Multi-stage `Dockerfile` + `Dockerfile.prod`, non-root user, healthchecks on django/postgres/redis.
- Nginx config per blueprint §13.3 (SSE location unbuffered).
- `scripts/backup_db.sh` + cron documentation per §13.5.
- README: quickstart that gets a fresh machine to a running, seeded app in ≤10 minutes.

## 5. Execution plan & phase gates

Follow blueprint §14. A phase is done only when its gate passes; stop at each gate and report what was built and how it was verified (commands + output), then continue.

- **Phase 1 — F1 Foundation** (tasks 1.1–1.11). Gate: `docker compose up` → home, hub, standings, calendar, results, round, qualifying, driver pages all render with real backfilled 2020→current data; `simulate_live` drives the live timing page end-to-end; dark/light + mobile verified; tests + CI green. *(Defer actual VPS deployment steps of 1.11 to docs: write `docs/DEPLOY.md` for Dokploy instead of touching a server.)*
- **Phase 2 — Accounts & Notifications** (2.1–2.6). Gate: signup/login (email + Google stub if no credentials), favourites, theme persistence, preference UI; reminder/result emails visible in dev console; push works in a local browser with generated VAPID keys; personalised feed renders.
- **Phase 3 — Sharing & Content** (3.1–3.5). Gate: finishing a simulated race auto-generates all five card types; card builder previews and downloads; WhatsApp link + caption copy work; FFmpeg renders a sample 15s clip on the dedicated `render` queue; public profile page live.
- **Phase 4 — Polish & Hardening** (4.1–4.8). Gate: admin dashboard + broadcast; scenario explorer; head-to-head; Chart.js visualisations; PWA installable with offline shell; Lighthouse ≥90 performance / ≥95 accessibility on home + standings; GDPR flows; SSE load test script (500 concurrent connections) documented with results.

## 6. Verification protocol (before every push)

1. `ruff format --check . && ruff check .`
2. `pytest`
3. `python manage.py makemigrations --check --dry-run` (no missing migrations)
4. `docker compose up -d` then smoke: `curl -f /healthz /  /f1/ /f1/standings/`
5. If the live layer was touched: run `simulate_live` and verify the SSE stream with `curl -N` + the browser UI.

## 7. Out of scope — do not build

Other-series ingest or pages (schema only), DRF or any public JSON API, mobile app, Django Channels/WebSockets, Node/npm tooling, Apple/Facebook login, i18n, fantasy/comments, Instagram Graph posting, real VPS provisioning.

## 8. Ask vs. assume

Ask the user **only** for things you cannot generate: Google OAuth credentials, SMTP credentials, production domain, deployment credentials. If unavailable, proceed with documented placeholders (`pitwall.example`, console email, Google provider configured but unverified) and leave a `docs/LAUNCH_CHECKLIST.md` listing exactly what to fill in. Generate VAPID keys yourself. Everything else: decide, log it in `docs/DECISIONS.md`, keep moving.

## 9. Definition of done (v1)

- All public, dashboard, and admin pages from blueprint §6 implemented and responsive in both themes.
- 2020→current F1 data backfilled; standings/calendar auto-refresh on schedule; live weekend path proven via simulator.
- Accounts, favourites, notifications (email + push), sharing cards, video export, scenario explorer, PWA, GDPR flows all working.
- Test suite and CI green; Lighthouse targets met; `docs/DEPLOY.md` + `docs/LAUNCH_CHECKLIST.md` complete; README quickstart verified from scratch.
