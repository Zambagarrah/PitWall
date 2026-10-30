# PitWall — Full Product & Technical Blueprint
> Unified Motorsport Stats Hub · Version 1.5 · October 2026

**Changes in v1.5:** idempotency UNIQUE keys on WRC/NASCAR stage results; footer version tag fixed.

**Changes in v1.4:** §8.5 deletion prose matches `notification_log` SET NULL; `.env.example` `REDIS_URL` uses localhost for out-of-Docker dev.

**Changes in v1.3:** active-pairing partial unique index + ingest flip rule, `notification_log` GDPR-safe FK, SSE fan-out snapshot iteration, `docs/` added to project tree.

**Changes in v1.2:** audit hardening — schema unique constraints (results, pit stops, laps), SSE hub crash-recovery + framing contract, ops/compose details, GDPR-safe FKs.

**Changes from v1.0:**
- **V1 scope is Formula 1 only.** The other eight series remain fully planned (schema, data sources, roadmap) and ship one at a time post-launch.
- **Frontend is Django-rendered templates** (HTMX + Alpine.js + Bootstrap 5). Next.js removed.
- **Real-time layer is Server-Sent Events (SSE)**, not Django Channels/WebSockets. Fewer moving parts, same UX at 5-second update rates.
- **Auth is email/password + Google only** at launch (session-based, via django-allauth). Apple/Facebook deferred.
- **Admin panel is Django's built-in admin**, extended with custom views only where needed.
- **Historical depth: 2020 → current season** backfilled from Jolpica.
- Full public REST API (DRF) deferred to the mobile-app phase; v1 serves HTML + SSE + a handful of JSON endpoints.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Product Requirements Document (PRD)](#2-product-requirements-document-prd)
3. [Data Sources & Real-Time Strategy](#3-data-sources--real-time-strategy)
4. [Technical Architecture](#4-technical-architecture)
5. [Database Schema](#5-database-schema)
6. [URL & View Design](#6-url--view-design)
7. [Real-Time Layer (SSE)](#7-real-time-layer-sse)
8. [User Accounts & Auth](#8-user-accounts--auth)
9. [Notifications System](#9-notifications-system)
10. [Social Sharing & Content Generation](#10-social-sharing--content-generation)
11. [Admin Panel](#11-admin-panel)
12. [Frontend Architecture (Django Templates)](#12-frontend-architecture-django-templates)
13. [Infrastructure & DevOps (Contabo + Dokploy)](#13-infrastructure--devops-contabo--dokploy)
14. [Feature Roadmap & Milestones](#14-feature-roadmap--milestones)
15. [Future: Public API & Mobile App](#15-future-public-api--mobile-app)

---

## 1. Executive Summary

**PitWall** is a public-facing, real-time motorsport statistics hub. **V1 launches with Formula 1 only** — live standings, race results, calendar, qualifying, pit stops, sector times, and lap-by-lap live timing updating within seconds during a race weekend. The architecture, database schema, and roadmap are built multi-series from day one: Formula 2, Formula 3, Formula E, WEC, WRC, NASCAR, MotoGP, and GT follow one at a time after launch.

### Core Value Propositions

- **One hub, all series (eventually)** — F1 first, the other eight championships added incrementally on the same platform.
- **Live during race weekends** — Lap-by-lap updates streamed via SSE, not page refreshes.
- **Personal experience** — User accounts with favourite drivers, saved series, and smart alerts.
- **Shareable** — Auto-generated race cards and custom graphic/video content for Instagram, WhatsApp, and TikTok.
- **Free at launch** — Fully open, no paywalls, monetisation explored later.

### Key Numbers (Design Targets)

| Metric | Target |
|---|---|
| Series covered at launch | 1 (F1) — 9 on the roadmap |
| Historical data at launch | F1 seasons 2020 → current |
| Live update frequency (race weekend) | Every 5–10 seconds |
| Notification channels | Email + Web Push |
| Platforms | Web (responsive), PWA-ready for future mobile |
| Hosting | Single Contabo VPS via Dokploy (Docker) |

---

## 2. Product Requirements Document (PRD)

### 2.1 Audience

- Motorsport fans (casual to enthusiast) worldwide
- Public-facing, no login required to browse
- Login unlocks personalization, notifications, and social sharing

### 2.2 Series Coverage

| Series | Divisions / Classes | Phase |
|---|---|---|
| Formula 1 | Drivers, Constructors | **V1 (launch)** |
| Formula 2 | Feature & Sprint races, Teams | Post-launch |
| Formula 3 | Feature & Sprint races, Teams | Post-launch |
| Formula E | Drivers, Teams, ABB FIA rounds | Post-launch |
| WEC | Hypercar, LMP2, GT3 classes | Post-launch |
| WRC | Drivers, Co-Drivers, Manufacturers, WRC2 | Post-launch |
| NASCAR | Cup Series, Xfinity, Trucks | Post-launch |
| MotoGP | MotoGP, Moto2, Moto3 | Post-launch |
| GT (GTWC, IGTC) | Pro, Pro-Am, Am classes | Post-launch |

### 2.3 Core Features (All Series)

#### Standings
- Driver/Rider standings with points, gaps, wins, podiums
- Constructor/Team standings
- Championship gap visualisation (bar chart + points delta to leader)

#### Race Results
- Full classified results per race/round
- Fastest lap holder and time
- DNF / DSQ / DNS reasons
- Grid vs finish position delta

#### Race Calendar
- Full season calendar per series
- Upcoming round countdown
- Round status: Upcoming / Live / Completed
- Session schedule per round (Practice, Quali, Race, Sprint)

#### Championship Scenarios
- Points needed to clinch / be eliminated
- "What if" scenario explorer (manual input)

### 2.4 Series-Specific Features

#### F1 (V1) / F2 / F3
- Qualifying gap to pole (Q1/Q2/Q3 breakdown for F1)
- Sector times per session (S1, S2, S3)
- Pit stop data: stop number, lap, duration, tyre compound fitted
- DRS zone map per circuit (static diagram)
- Sprint race vs main race points split

#### WEC / GT (post-launch)
- Class-based standings (Hypercar, LMP2, GT3 in WEC; Pro/Pro-Am/Am in GT)
- Team endurance stats: laps led, pit stop count, stints per driver
- Lap chart per class

#### WRC (post-launch)
- Stage-by-stage split times (SS1, SS2, …)
- Overall rally standings after each stage
- Power Stage top-5 bonus points
- Manufacturer standings

#### NASCAR (post-launch)
- Stage 1 / Stage 2 / Final stage points breakdown
- Playoff picture (Regular Season standings + Playoff bracket)
- Loop data: avg running position, quality passes, green-flag speed
- Stage caution history

#### MotoGP (post-launch)
- Q1 / Q2 qualifying split and who auto-progresses
- Sprint race (Saturday) vs Grand Prix (Sunday) separate points tables
- Tyre allocation stats
- Rider championship eligibility / wildcard status

### 2.5 User Account Features

| Feature | Detail |
|---|---|
| Favourite Drivers | Up to 10, shown in personalised feed |
| Favourite Series | Reorders homepage series cards (single series at launch) |
| Notification Preferences | Per-driver, per-series, per-event type |
| Shareable Profile | Public link with fan stats (series followed, events tracked) |
| Social Account Linking | Instagram, WhatsApp (share), TikTok |
| Social Login | **Google, Email/Password** (Apple/Facebook deferred) |
| Theme | Light / Dark (per-account, overridable per-session) |

### 2.6 Non-Functional Requirements

| Requirement | Spec |
|---|---|
| Uptime | 99.5% target (single VPS; no SLA-grade redundancy at launch) |
| Live update latency | < 15 seconds from source during race |
| Page load (FCP) | < 2 seconds on 4G mobile |
| Responsive breakpoints | 375px (mobile), 768px (tablet), 1280px (desktop) |
| Accessibility | WCAG 2.1 AA |
| Dark / Light mode | Both, respects system preference, user can override |
| GDPR / Privacy | Consent banner, data deletion on account removal |

---

## 3. Data Sources & Real-Time Strategy

### 3.1 Source Matrix

| Series | Primary Source | Secondary / Fallback | Live? | Phase |
|---|---|---|---|---|
| **F1** | OpenF1 API (free, live) | Jolpica / Ergast (historical), F1 official site | ✅ Lap-by-lap | **V1** |
| **F2 / F3** | FIA results PDFs + scraper | Jolpica (limited F2 data) | ⚠️ Post-session scrape | Post-launch |
| **Formula E** | FE official API (unofficial endpoints) | ABB FIA page scraper | ⚠️ Near-live | Post-launch |
| **WEC** | WEC official timing (fiawec.com) | ACO results scraper | ⚠️ Near-live | Post-launch |
| **WRC** | WRC+ API (unofficial) / ewrc-results.com | WRC official site scraper | ⚠️ Stage-by-stage | Post-launch |
| **NASCAR** | NASCAR Stats API (official, free) | RapidAPI NASCAR endpoints | ✅ Live loops | Post-launch |
| **MotoGP** | MotoGP official unofficial endpoints | RapidAPI MotoGP | ⚠️ Session-level | Post-launch |
| **GT / GTWC** | GTWC official results + scraper | RapidAPI | ⚠️ Post-session | Post-launch |

**Legend:** ✅ = true real-time (seconds), ⚠️ = near-real-time (minutes) or post-session

> **Risk note:** Non-F1 sources are largely unofficial endpoints and scrapers — fragile and ToS-sensitive. This is the main reason they ship one at a time post-launch, each behind its own ingest module that can fail without affecting the rest of the site.

### 3.2 V1 Data Plan (F1)

- **OpenF1** (`https://api.openf1.org/v1/`) — live session data: positions, intervals, laps, sectors, pit stops, stints, race control messages.
- **Jolpica / Ergast** (`https://api.jolpi.ca/ergast/`) — historical + post-race canonical data: seasons, rounds, results, qualifying, standings. **Backfill 2020 → current season** on first deploy.
- Reconciliation: live data from OpenF1 is treated as provisional; Jolpica results overwrite after each round is published.

### 3.3 Data Ingestion Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    DATA INGESTION LAYER                  │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  API Pollers │  │   Scrapers   │  │  Streaming   │  │
│  │  (per series)│  │ (post-launch │  │  (OpenF1)    │  │
│  │  Celery      │  │   series)    │  │              │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         └─────────────────▼──────────────────┘          │
│                    ┌──────────────┐                      │
│                    │ Celery Tasks │                      │
│                    └──────┬───────┘                      │
└───────────────────────────┼──────────────────────────────┘
                            │
              ┌─────────────▼──────────────┐
              │   Redis Pub/Sub + Cache     │
              └─────────────┬──────────────┘
                            │
              ┌─────────────▼──────────────┐
              │   PostgreSQL (persistent)  │
              └─────────────┬──────────────┘
                            │
              ┌─────────────▼──────────────┐
              │  Django (ASGI) — SSE views │
              │  + rendered templates      │
              └────────────────────────────┘
```

### 3.4 Polling Intervals

| Mode | Interval | Trigger |
|---|---|---|
| Race weekend (live session, incl. red-flag `paused`) | 5 seconds | Session state = LIVE / PAUSED |
| Race weekend (between sessions) | 60 seconds | Weekend mode: round live, no session live |
| Off-season / week between rounds | 15 minutes | Default |
| Post-race data (pit stops, sectors) | On-demand after chequered flag | Session state = FINISHED |

**Dispatcher pattern (instead of static 5s beat entries):** a lightweight `tick` task runs every minute via Celery Beat, checks session states in the DB, and sets a Redis flag when a session is LIVE. A self-rescheduling fast-poll task loops at 5s **only while the flag is set**, so nothing hammers external APIs off-season.

### 3.5 Management Commands & Ingest Logging

| Command | Purpose |
|---|---|
| `manage.py seed_series` | Idempotently creates the 9 `series` rows (only F1 `is_active`) and default `duration_mins` per session type (FP/Q = 60, sprint quali = 45…). Safe to re-run. |
| `manage.py backfill_f1 --from-season 2020 [--delay 0.3] [--resume]` | Backfills seasons → rounds → sessions → drivers → teams → entries → results → qualifying → standings from Jolpica. Throttled to **≤4 req/s and ≤500 req/h** (hard Jolpica limits); a full 2020→current backfill is ~1,000+ calls, so expect **2–3 hours**. Progress is checkpointed per round in `ingest_log`, so an interrupted run resumes instead of restarting. |
| `manage.py simulate_live <session_id> [--speed 10] [--source frames.json]` | Replays recorded OpenF1 frames (or synthetic data) through the **real** pipeline — writes `live_session_state`, publishes SSE frames to the Redis channel, refreshes the snapshot key — at `--speed`× real time. Built **with** the live layer, not after: it is the acceptance vehicle for all live features off-season. |

Every ingest run writes an `ingest_log` row (source, task, started/finished, status, message, items written). The latest success timestamp per source powers:
- the **admin health dashboard** (§11.3), and
- a public **"Data updated X ago" badge** on standings/results pages whenever the source is stale (> 2× its expected refresh interval) — source downtime serves stale DB data, never a 500.

---

## 4. Technical Architecture

### 4.1 Stack Summary

| Layer | Technology | Rationale |
|---|---|---|
| Web framework | Django 5 (ASGI via Uvicorn) | Batteries-included; templates, ORM, admin, auth |
| Frontend | Django templates + HTMX + Alpine.js + Bootstrap 5 | Server-rendered, SEO-native, no build step at all |
| Real-time | Server-Sent Events (async Django views + Redis pub/sub) | One-way push is all we need at 5s cadence |
| Async tasks | Celery + Celery Beat | Scheduled polling, background jobs |
| Cache / Pub-Sub | Redis 7 | Fragment/view cache, SSE message broker, Celery broker |
| Database | PostgreSQL 16 | Relational, robust, JSON support |
| Auth | django-allauth (sessions) | Email/password + Google; session cookies, CSRF-protected |
| Email | SMTP (Django email backend) | Transactional + notification emails |
| Push Notifications | Web Push (pywebpush) | Browser push without mobile app |
| Image Generation | Pillow | Server-side race result cards |
| Video/GIF export | FFmpeg (server-side) | TikTok/Reels clip assembly (Phase 3) |
| Charts | Chart.js (vendored static) | Standings gaps, lap charts |
| Containerisation | Docker + Docker Compose | Consistent deploys via Dokploy |
| Reverse Proxy | Nginx (inside Dokploy) | SSL, routing, static files |
| CI/CD | Dokploy Git integration | Auto-deploy on push |

### 4.2 High-Level System Diagram

```
Browser / Mobile
      │
      │ HTTPS (443)
      ▼
┌─────────────┐
│   Nginx     │  (Reverse Proxy / SSL Termination / static files)
└──────┬──────┘
       │
       ▼
┌──────────────────┐
│  Django (ASGI)   │  HTML pages + HTMX fragments + SSE streams
│  (Port 8000)     │
└────────┬─────────┘
         │
   ┌─────┼──────────────────────────┐
   │     │                          │
   ▼     ▼                          ▼
┌──────────┐   ┌──────────────┐   ┌──────────────┐
│PostgreSQL│   │ Redis Cache  │   │  Celery      │
│  (5432)  │   │  + Pub/Sub   │   │  Workers     │
└──────────┘   │  (6379)      │   │  + Beat      │
               └──────────────┘   └──────┬───────┘
                                         │
                                (External APIs +
                                   Scrapers)
```

### 4.3 Project Structure (Single Django Project)

```
pitwall/
├── manage.py
├── pitwall/                    # Django project root
│   ├── settings/
│   │   ├── base.py
│   │   ├── development.py
│   │   └── production.py
│   ├── urls.py
│   ├── asgi.py                 # Uvicorn entry point (ASGI-only deploy; no wsgi.py)
│   └── celery.py
├── apps/
│   ├── accounts/               # User model, allauth config, profiles
│   ├── series/                 # Series, Seasons, Rounds, Sessions
│   ├── drivers/                # Driver & team models
│   ├── results/                # Race results, standings
│   ├── live/                   # Live session state + SSE views
│   ├── notifications/          # Email + push notification
│   ├── sharing/                # Card generation
│   └── ingest/                 # Celery tasks + API clients (per series)
│       ├── clients/            # openf1.py, jolpica.py, (later: wrc.py, …)
│       ├── tasks/              # f1.py, dispatcher.py, (later per series)
│       └── management/commands/  # seed_series.py, backfill_f1.py, simulate_live.py
├── templates/
│   ├── base.html               # Theme, nav, htmx/alpine includes
│   ├── home.html               # Series cards overview
│   ├── series/                 # hub, standings, calendar, results
│   ├── rounds/                 # round detail, qualifying, race, live
│   ├── drivers/                # driver profile
│   ├── dashboard/              # personalised feed, favourites, prefs
│   ├── account/                # allauth template overrides
│   ├── emails/                 # notification email templates
│   └── partials/               # HTMX fragments (timing table, standings rows…)
├── static/
│   ├── css/                    # bootstrap.min.css + pitwall.css (theme overrides)
│   ├── js/                     # bootstrap.bundle.min.js, htmx.min.js, alpine.min.js, chart.min.js, sw.js
│   └── img/series-badges/      # generated placeholder SVGs only — never official logos
├── requirements/
│   ├── base.txt
│   └── production.txt
├── docs/
│   ├── BLUEPRINT.md            # this document — single source of truth
│   ├── DECISIONS.md            # one-line build decision log
│   ├── DEPLOY.md               # Dokploy deployment guide (Phase 1.11)
│   └── LAUNCH_CHECKLIST.md     # credentials/domain to fill in before launch
├── docker-compose.yml
├── docker-compose.prod.yml
└── .env.example
```

Bootstrap 5, HTMX, Alpine.js, and Chart.js are all vendored static files — no Node.js toolchain or CSS build step. Brand theming lives in `static/css/pitwall.css` as Bootstrap CSS-variable overrides (`--bs-primary`, series accents, dark-mode tweaks).

---

## 5. Database Schema

The schema is multi-series from day one. Series-specific tables (WRC stages, NASCAR stage points) are created up front but only populated when their series ships.

### 5.1 Core Tables

```sql
-- ─── SERIES & SEASONS ────────────────────────────────────────

CREATE TABLE series (
    id          SERIAL PRIMARY KEY,
    slug        VARCHAR(20) UNIQUE NOT NULL,  -- 'f1','f2','wec','wrc','nascar','motogp'
    name        VARCHAR(100) NOT NULL,
    short_name  VARCHAR(20),
    logo_url    VARCHAR(500),
    is_active   BOOLEAN DEFAULT true
);

CREATE TABLE seasons (
    id          SERIAL PRIMARY KEY,
    series_id   INTEGER REFERENCES series(id),
    year        INTEGER NOT NULL,
    is_current  BOOLEAN DEFAULT false,
    UNIQUE(series_id, year)
);

-- ─── ROUNDS & SESSIONS ───────────────────────────────────────

CREATE TABLE rounds (
    id              SERIAL PRIMARY KEY,
    season_id       INTEGER REFERENCES seasons(id),
    round_number    INTEGER NOT NULL,
    name            VARCHAR(200),           -- "British Grand Prix"
    circuit_name    VARCHAR(200),
    country         VARCHAR(100),
    city            VARCHAR(100),
    status          VARCHAR(20) DEFAULT 'upcoming',  -- upcoming|live|completed|cancelled|postponed
    round_start     TIMESTAMPTZ,
    round_end       TIMESTAMPTZ,
    UNIQUE(season_id, round_number)
);

CREATE TABLE sessions (
    id              SERIAL PRIMARY KEY,
    round_id        INTEGER REFERENCES rounds(id),
    session_type    VARCHAR(30),  -- practice1|practice2|qualifying|sprint_quali|sprint|race
    scheduled_at    TIMESTAMPTZ,
    status          VARCHAR(20) DEFAULT 'upcoming',  -- upcoming|live|paused|finished|cancelled|postponed
    duration_mins   INTEGER        -- seeded with per-type defaults (FP/Q = 60, sprint quali = 45, …)
);

-- ─── DRIVERS & TEAMS ─────────────────────────────────────────

-- NOTE: team identity is deliberately per-series. Cross-series career views
-- aggregate by driver, not team; add a canonical team grouping post-launch
-- only if a cross-series team page is ever needed.
CREATE TABLE teams (
    id              SERIAL PRIMARY KEY,
    series_id       INTEGER REFERENCES series(id),
    name            VARCHAR(200),
    short_name      VARCHAR(50),
    nationality     VARCHAR(100),
    logo_url        VARCHAR(500),
    primary_color   VARCHAR(7),
    secondary_color VARCHAR(7)
);

CREATE TABLE drivers (
    id              SERIAL PRIMARY KEY,
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    slug            VARCHAR(150) UNIQUE,
    nationality     VARCHAR(100),
    date_of_birth   DATE,
    number          INTEGER,
    abbreviation    VARCHAR(3),
    photo_url       VARCHAR(500),
    bio             TEXT
);

CREATE TABLE driver_season_entries (
    id          SERIAL PRIMARY KEY,
    driver_id   INTEGER REFERENCES drivers(id),
    team_id     INTEGER REFERENCES teams(id),
    season_id   INTEGER REFERENCES seasons(id),
    car_number  INTEGER,
    is_active   BOOLEAN DEFAULT true,
    -- One row per driver-team pairing per season. Mid-season swaps = one row
    -- per team; A→B→A needs no third row because race_results snapshots the
    -- team actually raced for. is_active marks the current pairing: ingest
    -- MUST set is_active = false on all existing (driver_id, season_id) rows
    -- before inserting/re-activating the new one (enforced by the partial
    -- unique index in §5.2).
    UNIQUE(driver_id, season_id, team_id)
);

-- ─── STANDINGS ───────────────────────────────────────────────

CREATE TABLE driver_standings (
    id              SERIAL PRIMARY KEY,
    season_id       INTEGER REFERENCES seasons(id),
    round_id        INTEGER REFERENCES rounds(id),  -- NULL = current
    driver_id       INTEGER REFERENCES drivers(id),
    team_id         INTEGER REFERENCES teams(id),
    position        INTEGER,
    points          NUMERIC(8,2),
    wins            INTEGER DEFAULT 0,
    podiums         INTEGER DEFAULT 0,
    poles           INTEGER DEFAULT 0,
    fastest_laps    INTEGER DEFAULT 0,
    class           VARCHAR(50),   -- for WEC/GT class splits
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    -- PG16: one row per driver per snapshot; round_id NULL = current.
    -- Required for update_or_create to update instead of appending.
    UNIQUE NULLS NOT DISTINCT (season_id, round_id, driver_id)
);

CREATE TABLE team_standings (
    id              SERIAL PRIMARY KEY,
    season_id       INTEGER REFERENCES seasons(id),
    round_id        INTEGER REFERENCES rounds(id),
    team_id         INTEGER REFERENCES teams(id),
    position        INTEGER,
    points          NUMERIC(8,2),
    wins            INTEGER DEFAULT 0,
    class           VARCHAR(50),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE NULLS NOT DISTINCT (season_id, round_id, team_id)
);

-- ─── RACE RESULTS ────────────────────────────────────────────

CREATE TABLE race_results (
    id              SERIAL PRIMARY KEY,
    session_id      INTEGER REFERENCES sessions(id),
    driver_id       INTEGER REFERENCES drivers(id),
    team_id         INTEGER REFERENCES teams(id),
    position        INTEGER,
    grid_position   INTEGER,
    status          VARCHAR(50),   -- Finished | DNF | DSQ | DNS
    points          NUMERIC(6,2),
    laps_completed  INTEGER,
    finish_time     VARCHAR(30),   -- "1:23:45.678" or "+4.123"
    fastest_lap     BOOLEAN DEFAULT false,
    fastest_lap_time VARCHAR(20),
    fastest_lap_number INTEGER,
    class           VARCHAR(50),
    UNIQUE(session_id, driver_id)  -- idempotent ingest: update_or_create key
);

-- ─── F1-SPECIFIC: PIT STOPS ──────────────────────────────────

CREATE TABLE pit_stops (
    id              SERIAL PRIMARY KEY,
    session_id      INTEGER REFERENCES sessions(id),
    driver_id       INTEGER REFERENCES drivers(id),
    stop_number     INTEGER,
    lap             INTEGER,
    duration_ms     INTEGER,
    compound_in     VARCHAR(20),  -- SOFT|MEDIUM|HARD|INTER|WET
    compound_out    VARCHAR(20),
    UNIQUE(session_id, driver_id, stop_number)  -- idempotent ingest
);

-- ─── F1-SPECIFIC: SECTOR / LAP TIMES ────────────────────────

CREATE TABLE lap_times (
    id              SERIAL PRIMARY KEY,
    session_id      INTEGER REFERENCES sessions(id),
    driver_id       INTEGER REFERENCES drivers(id),
    lap_number      INTEGER,
    lap_duration_ms INTEGER,
    sector1_ms      INTEGER,
    sector2_ms      INTEGER,
    sector3_ms      INTEGER,
    compound        VARCHAR(20),
    is_personal_best BOOLEAN DEFAULT false,
    is_session_best  BOOLEAN DEFAULT false,
    UNIQUE(session_id, driver_id, lap_number)  -- idempotent ingest
);
-- lap_times grows fast: ~1,100 rows per race, ~150k+ over a 5-season backfill
-- before practice/quali laps. The (session_id, driver_id) index below is
-- critical; partition by season if it ever becomes a problem.

-- ─── WRC-SPECIFIC: STAGES (populated post-launch) ───────────

CREATE TABLE wrc_stages (
    id              SERIAL PRIMARY KEY,
    round_id        INTEGER REFERENCES rounds(id),
    stage_number    INTEGER,
    name            VARCHAR(100),  -- "SS1 - Oulton Park"
    distance_km     NUMERIC(6,2),
    is_power_stage  BOOLEAN DEFAULT false,
    status          VARCHAR(20) DEFAULT 'upcoming'
);

CREATE TABLE wrc_stage_results (
    id              SERIAL PRIMARY KEY,
    stage_id        INTEGER REFERENCES wrc_stages(id),
    driver_id       INTEGER REFERENCES drivers(id),
    stage_time_ms   BIGINT,
    overall_time_ms BIGINT,
    stage_position  INTEGER,
    overall_position INTEGER,
    gap_to_leader_ms BIGINT,
    power_stage_points INTEGER DEFAULT 0,
    UNIQUE(stage_id, driver_id)
);

-- ─── NASCAR-SPECIFIC: STAGE POINTS (populated post-launch) ──

CREATE TABLE nascar_stage_results (
    id              SERIAL PRIMARY KEY,
    session_id      INTEGER REFERENCES sessions(id),
    driver_id       INTEGER REFERENCES drivers(id),
    stage_number    INTEGER,   -- 1, 2, 3
    stage_position  INTEGER,
    stage_points    INTEGER,
    laps_led        INTEGER DEFAULT 0,
    UNIQUE(session_id, driver_id, stage_number)
);

-- ─── LIVE SESSION STATE ──────────────────────────────────────

CREATE TABLE live_session_state (
    id              SERIAL PRIMARY KEY,
    session_id      INTEGER UNIQUE REFERENCES sessions(id),
    current_lap     INTEGER,
    total_laps      INTEGER,
    session_elapsed_ms INTEGER,    -- elapsed ms (sortable/computable; formatted in templates)
    safety_car      BOOLEAN DEFAULT false,
    virtual_sc      BOOLEAN DEFAULT false,
    red_flag        BOOLEAN DEFAULT false,
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- ─── USER ACCOUNTS ───────────────────────────────────────────
-- Implemented as a custom Django user model; allauth handles
-- social accounts (its own tables). Shown here conceptually.

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               VARCHAR(254) UNIQUE NOT NULL,
    username            VARCHAR(50) UNIQUE,
    display_name        VARCHAR(100),
    password_hash       VARCHAR(255),
    avatar_url          VARCHAR(500),
    role                VARCHAR(20) DEFAULT 'user',  -- user|moderator|admin
    email_verified      BOOLEAN DEFAULT false,
    theme               VARCHAR(10) DEFAULT 'system',  -- light|dark|system
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    last_login          TIMESTAMPTZ
);

CREATE TABLE user_favourites (
    id          SERIAL PRIMARY KEY,
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    entity_type VARCHAR(20),   -- driver|team|series
    entity_id   INTEGER,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, entity_type, entity_id)
);

-- ─── NOTIFICATIONS ───────────────────────────────────────────

CREATE TABLE notification_preferences (
    id              SERIAL PRIMARY KEY,
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    channel         VARCHAR(20),  -- email|push
    event_type      VARCHAR(50),  -- session_start|race_result|standings_update|driver_news
    entity_type     VARCHAR(20),  -- driver|series|all
    entity_id       INTEGER,
    is_enabled      BOOLEAN DEFAULT true
);

CREATE TABLE push_subscriptions (
    id              SERIAL PRIMARY KEY,
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    endpoint        TEXT NOT NULL UNIQUE,  -- re-subscribes update_or_create on endpoint; no dupes
    p256dh          TEXT,
    auth            TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE notification_log (
    id              SERIAL PRIMARY KEY,
    user_id         UUID REFERENCES users(id) ON DELETE SET NULL,  -- keep de-identified log rows; GDPR delete must not fail
    channel         VARCHAR(20),
    event_type      VARCHAR(50),
    subject         VARCHAR(300),
    body            TEXT,
    sent_at         TIMESTAMPTZ DEFAULT NOW(),
    status          VARCHAR(20) DEFAULT 'sent'
);

-- ─── OPERATIONS ──────────────────────────────────────────────

CREATE TABLE ingest_log (
    id              SERIAL PRIMARY KEY,
    source          VARCHAR(30),       -- openf1|jolpica|…
    task            VARCHAR(100),
    started_at      TIMESTAMPTZ DEFAULT NOW(),
    finished_at     TIMESTAMPTZ,
    status          VARCHAR(20) DEFAULT 'running',  -- running|success|error
    message         TEXT,              -- error detail / resume checkpoint
    items_written   INTEGER DEFAULT 0
);

CREATE TABLE announcements (
    id          SERIAL PRIMARY KEY,
    title       VARCHAR(200),
    body        TEXT,
    is_active   BOOLEAN DEFAULT false,
    starts_at   TIMESTAMPTZ,
    ends_at     TIMESTAMPTZ,
    created_by  UUID REFERENCES users(id) ON DELETE SET NULL  -- GDPR deletion must never be blocked
);
```

### 5.2 Indexes

```sql
-- Performance indexes for hot query paths
CREATE INDEX idx_driver_standings_season ON driver_standings(season_id);
CREATE INDEX idx_race_results_session ON race_results(session_id);
CREATE INDEX idx_lap_times_session_driver ON lap_times(session_id, driver_id);
CREATE INDEX idx_user_favourites_user ON user_favourites(user_id);
CREATE INDEX idx_notification_prefs_user ON notification_preferences(user_id);
CREATE INDEX idx_live_state_session ON live_session_state(session_id);
CREATE INDEX idx_notif_log_sent_at ON notification_log(sent_at DESC);
CREATE INDEX idx_notif_log_status ON notification_log(status);
CREATE INDEX idx_ingest_log_source_time ON ingest_log(source, started_at DESC);

-- At most one active team pairing per driver per season (see §5.1 entries note)
CREATE UNIQUE INDEX uq_driver_season_active
    ON driver_season_entries(driver_id, season_id) WHERE is_active;
```

---

## 6. URL & View Design

V1 serves **HTML pages and HTMX fragments**, not a public JSON API. (The DRF API ships with the mobile-app phase — see §15.) A few JSON endpoints exist where a form/fragment doesn't fit (push subscription, card generation).

### 6.1 Public Pages (No Auth Required)

```
GET  /                                → Home: series cards overview
GET  /<series>/                       → Series hub (standings summary + next round)
GET  /<series>/standings/             → Full driver standings table
GET  /<series>/standings/teams/       → Team/constructor standings
GET  /<series>/calendar/              → Season calendar (?season=YYYY)
GET  /<series>/results/               → Results list, all rounds
GET  /<series>/rounds/<n>/            → Round overview + session schedule
GET  /<series>/rounds/<n>/race/       → Race result + pit stops + fastest laps
GET  /<series>/rounds/<n>/qualifying/ → Qualifying result (Q1/Q2/Q3)
GET  /<series>/rounds/<n>/live/       → Live timing page (SSE-driven)
GET  /drivers/<slug>/                 → Driver profile + career stats
GET  /privacy/                        → Privacy policy (GDPR)
GET  /robots.txt                      → robots.txt (template-rendered)
GET  /sitemap.xml                     → django.contrib.sitemaps
GET  /healthz                         → JSON health: DB + Redis ping (Docker healthchecks)
```

### 6.2 HTMX Fragment Endpoints (partials)

```
GET  /partials/<series>/standings-table/      → <table> fragment
GET  /partials/<series>/next-round/           → countdown card fragment
GET  /partials/live/<session_id>/timing/      → live timing <tbody> fragment
GET  /partials/rounds/<n>/laptimes-chart/     → lap chart fragment
```

### 6.3 Authenticated Pages & Endpoints

```
/accounts/…                           → allauth: login, signup, logout,
                                        password reset, Google OAuth
GET   /dashboard/                     → Personalised feed
GET   /dashboard/favourites/          → Manage favourites
POST  /dashboard/favourites/add/      → (HTMX form post)
POST  /dashboard/favourites/remove/
GET   /dashboard/notifications/       → Notification preferences UI
POST  /dashboard/notifications/       → Save preferences
GET   /dashboard/share/<session_id>/  → Card builder
POST  /dashboard/share/generate/      → JSON: trigger card/video generation
GET   /settings/                      → Profile, theme
POST  /push/subscribe/                → JSON: store Web Push subscription
POST  /push/unsubscribe/              → JSON
```

### 6.4 SSE Endpoints

```
GET  /sse/live/<session_id>/          → text/event-stream: live timing
GET  /sse/standings/<series>/         → text/event-stream: standings changes
GET  /sse/alerts/                     → text/event-stream: personal alerts (auth)
```

### 6.5 Admin

```
/admin/                               → Django built-in admin (staff only)
/admin/dashboard/                     → Custom view: ingest health, live stats
/admin/broadcast/                     → Custom view: compose broadcast notification
```

---

## 7. Real-Time Layer (SSE)

### 7.1 Why SSE over WebSockets

Live timing is strictly **server → client** at a 5-second cadence. SSE delivers that over plain HTTP with built-in auto-reconnect (`Last-Event-ID`), works through proxies, and removes Django Channels, the channel layer, and a separate WS client from the stack. If sub-second bidirectional needs ever appear (chat, predictions), Channels can be added then.

### 7.2 Architecture

```
Browser (EventSource / htmx sse extension)
      │
      │ GET /sse/live/42/   (text/event-stream, held open)
      ▼
Django ASGI (Uvicorn) — async view
      │
      │ SUBSCRIBE live:42
      ▼
Redis Pub/Sub
      ▲
      │ PUBLISH live:42 {...}
Celery Worker (every 5s during live session:
  poll OpenF1 → write DB → publish payload)
```

### 7.3 Event Types (Server → Client)

Same payloads as v1.0, delivered as named SSE events:

```
event: timing_update
data: {"session_id":42,"lap":35,"total_laps":57,"safety_car":false,
       "timing":[{"position":1,"driver_abbr":"NOR","gap":"LEADER",
                  "interval":"+0.000","last_lap":"1:21.456",
                  "sector1":"28.123","sector2":"31.456","sector3":"21.877",
                  "compound":"SOFT","pit_stops":1,"drs":true}, …]}

event: flag_change
data: {"flag":"SAFETY_CAR","message":"Safety car deployed — incident at Turn 12","lap":35}

event: standings_update
data: {"series":"f1","season":2026,
       "changes":[{"driver":"NOR","old_position":2,"new_position":1,"points":319}]}

event: personal_alert
data: {"message":"Max Verstappen just set fastest lap (Lap 44)",
       "event_type":"fastest_lap","driver_slug":"max-verstappen"}
```

`timing_update`, `flag_change`, and `standings_update` broadcast on shared channels (`live:<id>`, `standings:<series>`). `personal_alert` is **never** mixed into shared streams — it publishes to a per-user channel (`user:<id>:alerts`) and is delivered only on the authenticated `/sse/alerts/` stream.

### 7.4 Async SSE View (Simplified)

```python
# apps/live/sse.py — per-process broadcast hub: ONE Redis pub/sub connection
# per Uvicorn worker fans out to every local SSE client (O(workers) Redis
# connections, not O(viewers)).
import asyncio
import redis.asyncio as aioredis
from django.conf import settings

_redis = None
_subscribers: dict[str, set[asyncio.Queue]] = {}
_listener_task = None

def get_redis():
    global _redis
    if _redis is None:
        _redis = aioredis.from_url(settings.REDIS_URL)  # pooled, shared per worker
    return _redis

async def _listener():
    pubsub = get_redis().pubsub()
    await pubsub.psubscribe("live:*", "standings:*", "user:*")
    async for msg in pubsub.listen():
        if msg["type"] == "pmessage":
            # Snapshot: guards against subscriber-set mutation if an await is
            # ever added inside this loop body.
            for q in list(_subscribers.get(msg["channel"].decode(), ())):
                q.put_nowait(msg["data"].decode())

def _clear_listener(task: asyncio.Task):
    # If the listener dies (e.g. Redis blip), clear the handle so the next
    # subscribe() restarts it — otherwise SSE silently goes dead in this worker.
    global _listener_task
    _listener_task = None

async def subscribe(channel: str) -> asyncio.Queue:
    global _listener_task
    if _listener_task is None or _listener_task.done():
        _listener_task = asyncio.get_running_loop().create_task(_listener())
        _listener_task.add_done_callback(_clear_listener)
    q = asyncio.Queue()
    _subscribers.setdefault(channel, set()).add(q)
    return q

def unsubscribe(channel: str, q: asyncio.Queue):
    _subscribers.get(channel, set()).discard(q)
```

```python
# apps/live/views.py
import asyncio
from django.http import StreamingHttpResponse
from .sse import get_redis, subscribe, unsubscribe

async def live_session_stream(request, session_id: int):
    channel = f"live:{session_id}"

    async def event_stream():
        q = await subscribe(channel)
        try:
            # Initial snapshot so late joiners aren't blank
            snapshot = await get_redis().get(f"{channel}:snapshot")
            if snapshot:
                yield f"event: timing_update\ndata: {snapshot.decode()}\n\n"
            while True:
                try:
                    frame = await asyncio.wait_for(q.get(), timeout=15)
                    yield frame + "\n\n"  # frames arrive WITHOUT trailing newlines (see contract below)
                except asyncio.TimeoutError:
                    yield ": ping\n\n"  # 15s heartbeat — survives nginx/mobile idle timeouts
        finally:
            unsubscribe(channel, q)

    response = StreamingHttpResponse(event_stream(), content_type="text/event-stream")
    response["Cache-Control"] = "no-cache"
    response["X-Accel-Buffering"] = "no"   # disable nginx buffering
    return response
```

**Framing contract (publisher ↔ view):** the Celery poller publishes complete SSE frames **without trailing newlines** (`event: timing_update\ndata: {…}`); the view appends exactly one `\n\n` terminator. Never publish frames with trailing newlines — an extra blank line splits one event into two in the browser. The snapshot key stores only the bare JSON payload of the last `timing_update`; the view adds full framing when serving it. Because Redis pub/sub broadcasts to **every** subscriber, the hub makes multi-worker Uvicorn safe: each worker holds exactly one pub/sub connection regardless of viewer count.

### 7.5 Client Side (HTMX SSE extension)

```html
<!-- rounds/live.html -->
<div hx-ext="sse" sse-connect="/sse/live/{{ session.id }}/">
  <div sse-swap="flag_change" hx-swap="innerHTML">…flag banner…</div>
  <tbody sse-swap="timing_update" hx-swap="innerHTML">
    {% include "partials/timing_rows.html" %}
  </tbody>
</div>
```

For richer handling (e.g., Chart.js updates), a small Alpine.js store consumes the same `EventSource`. Reconnection is native to `EventSource` — no custom reconnect code needed.

---

## 8. User Accounts & Auth

### 8.1 Auth Model

Same-origin server-rendered app ⇒ **Django session authentication** (no JWT needed):

```
1. Register / Login → django-allauth (email/password or Google OAuth)
2. Session cookie issued (httpOnly, Secure, SameSite=Lax)
3. CSRF protection on all POSTs (Django middleware, htmx sends the token)
4. SSE /sse/alerts/ authenticates via the same session cookie
```

### 8.2 Login Providers (V1)

| Provider | Library | Setup Needed |
|---|---|---|
| Email / Password | django-allauth | — (email verification via SMTP) |
| Google | django-allauth[socialaccount] google provider | Google Cloud Console OAuth app |

**Deferred:** Apple (requires $99/yr developer account), Facebook (requires Meta app review). Both slot into allauth later without schema changes.

### 8.3 Allauth Configuration (settings/base.py excerpt)

```python
INSTALLED_APPS += [
    'allauth',
    'allauth.account',
    'allauth.socialaccount',
    'allauth.socialaccount.providers.google',
]

ACCOUNT_LOGIN_METHODS = {'email'}
ACCOUNT_SIGNUP_FIELDS = ['email*', 'password1*', 'password2*']
ACCOUNT_EMAIL_VERIFICATION = 'mandatory'

SOCIALACCOUNT_PROVIDERS = {
    'google': {
        'SCOPE': ['profile', 'email'],
        'AUTH_PARAMS': {'access_type': 'online'},
    },
}

# Brute-force protection — allauth's built-in rate limiting (no extra deps)
ACCOUNT_RATE_LIMITS = {
    'login_failed': '5/5m/ip',
    'signup': '10/h/ip',
    'reset_password': '5/h/ip',
}
```

### 8.4 User Roles

| Role | Permissions |
|---|---|
| `user` | Browse, personalise, notifications, share |
| `moderator` | View reports, hide/flag content (maps to Django `is_staff` + group) |
| `admin` | Django admin, data management, broadcast notifications (`is_superuser` or admin group) |

### 8.5 Privacy & GDPR

- **Cookie consent banner:** first-visit banner (small Alpine component in `base.html`; choice stored in a `pw_consent` cookie + localStorage). V1 sets only strictly-necessary cookies (session, CSRF, theme), so the banner is informational with a single "Got it" action until analytics are ever added. The web-push permission prompt is shown only after an explicit user gesture, never on page load.
- **Privacy page:** `/privacy/` template stub — data collected, retention, contact.
- **Account deletion:** self-service in `/settings/` — favourites, preferences, and push subscriptions cascade via `ON DELETE CASCADE`; notification log rows are de-identified (`user_id` set to NULL, preserving the audit trail); then the user row is deleted.

---

## 9. Notifications System

### 9.1 Event Types

| Event Type | Trigger | Channel |
|---|---|---|
| `session_reminder` | 30 min before session | Email + Push |
| `session_start` | Session goes LIVE | Push |
| `chequered_flag` | Race session ends | Push |
| `race_result_published` | Final results posted | Email + Push |
| `standings_change` | Positions change after race | Email + Push |
| `fastest_lap` | Favourite driver sets fastest lap | Push |
| `driver_news` | Admin posts driver news | Email + Push |
| `admin_broadcast` | Admin sends to all users | Email + Push |

### 9.2 Email (SMTP)

```python
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = env('SMTP_HOST')
EMAIL_PORT = env('SMTP_PORT', default=587)
EMAIL_USE_TLS = True
EMAIL_HOST_USER = env('SMTP_USER')
EMAIL_HOST_PASSWORD = env('SMTP_PASSWORD')
DEFAULT_FROM_EMAIL = 'PitWall <hello@pitwall.com>'
```

Email templates (HTML, Django templating) in `templates/emails/`:
- `session_reminder.html` — round info, session time, driver to watch
- `race_result.html` — top 10, fastest lap, standings impact
- `welcome.html` — onboarding with favourite series setup CTA

### 9.3 Web Push (pywebpush)

```python
# apps/notifications/push.py
from pywebpush import webpush, WebPushException
import json

def send_push(subscription, title, body, url=None, icon=None):
    """Call only from Celery tasks — blocking HTTP (requests); never from async views."""
    try:
        webpush(
            subscription_info={
                "endpoint": subscription.endpoint,
                "keys": {"p256dh": subscription.p256dh, "auth": subscription.auth},
            },
            data=json.dumps({
                "title": title,
                "body": body,
                "url": url or "https://pitwall.com",
                "icon": icon or "/static/icons/pitwall-192.png",
            }),
            vapid_private_key=settings.VAPID_PRIVATE_KEY,
            vapid_claims={"sub": f"mailto:{settings.VAPID_EMAIL}"},
        )
    except WebPushException as e:
        if e.response is not None and e.response.status_code in (404, 410):
            subscription.delete()  # endpoint expired or gone
```

The service worker (`static/js/sw.js`) handles `push` + `notificationclick`, and doubles as the PWA offline worker in Phase 4.

### 9.4 Celery Notification Tasks

```python
# apps/notifications/tasks.py
from celery import shared_task

@shared_task
def send_session_reminders():
    """Run every minute; find sessions starting in 30 min."""

@shared_task
def dispatch_race_result_notifications(session_id):
    """Triggered after chequered flag is detected."""

@shared_task
def send_broadcast(admin_message_id):
    """Admin-triggered broadcast to all subscribed users."""
```

---

## 10. Social Sharing & Content Generation

### 10.1 Auto-Generated Race Cards

When a race session is marked FINISHED, a Celery task generates shareable image cards with **Pillow**. Cards are stored in `/media/cards/` and served by Nginx.

| Card | Contents | Dimensions |
|---|---|---|
| Race Winner | Winner, team, time, round info | 1080×1080 (IG Square) |
| Top 3 Podium | P1/P2/P3 portraits, points gained | 1080×1080 |
| Race Result | Full top 10, fastest lap highlight | 1080×1920 (Story) |
| Championship Update | Standings after race, gap to leader | 1080×1080 |
| Driver Spotlight | Favourite driver's result + stats | 1080×1080 |

```python
# apps/sharing/generators.py
from django.conf import settings
from PIL import Image, ImageDraw, ImageFont

def generate_race_winner_card(session_id):
    result = RaceResult.objects.get(session_id=session_id, position=1)
    # Absolute paths only — CWD is unpredictable inside containers
    template = settings.BASE_DIR / 'apps/sharing/card_templates/race_winner_bg.png'
    img = Image.open(template)
    draw = ImageDraw.Draw(img)
    # Draw driver name, time, team colour strip, round name
    out_dir = settings.MEDIA_ROOT / 'cards'
    out_dir.mkdir(parents=True, exist_ok=True)
    path = out_dir / f'race_{session_id}_winner.webp'
    img.save(path, 'WEBP', quality=85)
    return f'{settings.MEDIA_URL}cards/{path.name}'
```

### 10.2 User-Triggered Custom Cards

The card builder page (`/dashboard/share/<session_id>/`) lets users:
1. Pick stats/moments to feature (lap time chart, head-to-head, personal highlight)
2. Preview live (Alpine.js-driven form → HTMX preview fragment)
3. Download or share directly

The form POSTs to `/dashboard/share/generate/`; the backend returns the generated image URL.

### 10.3 WhatsApp Sharing

No posting API — share via URL scheme:

```
https://wa.me/?text=<encoded_text_with_link>
```

For status (image), users download the card and upload manually. A "Copy for WhatsApp" button pre-formats the text.

### 10.4 Instagram Sharing

No general publishing API for web apps. PitWall provides:
- **Download card** — user posts manually
- **Copy caption** — auto-generated caption with hashtags
- Future: Instagram Business accounts via Graph API (`POST /me/media`)

### 10.5 TikTok / Reels Short Video (Phase 3)

Users select a result table, a stat to animate, and a duration (15s / 30s). The backend uses **FFmpeg** to assemble background (Pillow) + animated stat frames + optional royalty-free audio into 1080×1920 `.mp4`.

```bash
ffmpeg -loop 1 -i bg_frame.png \
       -i stats_overlay_%04d.png \
       -filter_complex "[0:v][1:v]overlay" \
       -c:v libx264 -t 15 -r 30 \
       output.mp4
```

> Video rendering is CPU-heavy on a single VPS — the `celery_render` service runs `-c 1 -Q render` with hard caps (`cpus: "1.0"`, `mem_limit: 1g`) so it never competes with live polling during a race.

---

## 11. Admin Panel

### 11.1 Approach

**Django's built-in admin** is the v1 admin panel — it covers data CRUD, user management, and result overrides for free. Custom staff-only views are added only where the admin falls short.

### 11.2 Built-in Admin Coverage (ModelAdmin customisation)

- Series / Seasons / Rounds / Sessions CRUD, with actions:
  - "Mark session LIVE / FINISHED"
  - "Trigger data refresh" (enqueues Celery task)
- Results: inline editing for corrections, bulk import form for missing data
- Users: search by email/username, promote to moderator/admin (groups), suspend (is_active), GDPR delete
- Notification log: read-only list with filters (sent/failed, channel)
- Push subscriptions: read-only

### 11.3 Custom Staff Views

| View | Purpose |
|---|---|
| `/admin/dashboard/` | Site stats (active users 24h, registrations, live SSE connection count), ingest health (last successful pull per series, error log tail), next 48h sessions |
| `/admin/broadcast/` | Compose + preview + send broadcast notification (email + push) |
| Announcements | Sitewide banner CRUD (admin model) + optional push |

Access control: `@staff_member_required` plus a `ModeratorRequiredMixin` for shared moderator/admin views.

---

## 12. Frontend Architecture (Django Templates)

### 12.1 Template Structure

```
templates/
├── base.html                 # <html>, theme class, nav, footer,
│                             #   htmx + alpine + chart.js includes
├── home.html                 # Series cards overview
├── series/
│   ├── hub.html              # Standings summary + next round + latest result
│   ├── standings.html        # Full standings (drivers/teams tabs)
│   ├── calendar.html         # Season calendar + countdown
│   └── results.html          # All rounds result list
├── rounds/
│   ├── detail.html           # Round overview + session schedule
│   ├── race.html             # Race result, pit stops, fastest laps
│   ├── qualifying.html       # Q1/Q2/Q3 breakdown
│   └── live.html             # Live timing (SSE-driven)
├── drivers/
│   └── profile.html          # Bio, career stats, season-by-season
├── dashboard/
│   ├── feed.html             # Personalised feed
│   ├── favourites.html
│   ├── notifications.html    # Preference matrix
│   └── share.html            # Card builder
├── account/                  # allauth overrides (login, signup…)
├── emails/                   # HTML email templates
└── partials/                 # HTMX fragments
    ├── standings_table.html
    ├── timing_rows.html
    ├── next_round_card.html
    └── flag_banner.html
```

### 12.2 Frontend Toolkit

| Tool | Role |
|---|---|
| **Bootstrap 5.3** | All styling + components (navbar, tabs, dropdowns, modals, toasts); vendored CSS/JS, customised via CSS-variable overrides in `pitwall.css` |
| **HTMX** | Partial page updates, form posts, SSE extension for live fragments |
| **Alpine.js** | Local interactivity: theme toggle, countdowns, card-builder state (Bootstrap JS covers standard components) |
| **Chart.js** | Championship gap bars, lap time charts, position progression |
| **django-htmx** | Middleware + request helpers (`request.htmx`) |

### 12.3 Design System

- **Typography:** Inter (sans) + JetBrains Mono (timing/numbers)
- **Theme:** Bootstrap 5.3 colour modes — `data-bs-theme="dark"` on `<html>`; Alpine toggle persisted to localStorage + user profile; respects `prefers-color-scheme` by default
- **Team/series colours:** per-team CSS custom properties injected from DB values
- **Live indicators:** pulsing red dot for LIVE sessions, amber for Safety Car
- **Series identity:** each series gets its own accent colour (see Appendix A)

### 12.4 Caching Strategy (replaces ISR)

Per-view / fragment caching in Redis via Django's cache framework:

| Page | Cache TTL | Notes |
|---|---|---|
| Home / Series hub | 60s | `cache_page` |
| Standings | 30s | Invalidated by ingest task on standings write |
| Calendar | 1 hour | Invalidated on calendar ingest |
| Round results | 10s during/post-race, 1h once round completed | Status-aware TTL |
| Live timing page | No cache | Shell is cheap; data arrives via SSE |
| Driver profile | 1 hour | |
| Dashboard / settings | No cache | Per-user |
| HTMX fragments | Template fragment cache (`{% cache %}`) keyed per series/session | |

SEO: server-rendered HTML is fully crawlable out of the box; add sitemap (`django.contrib.sitemaps`) + meta/OG tags per page.

---

## 13. Infrastructure & DevOps (Contabo + Dokploy)

### 13.1 Server Layout (Single VPS)

```
Contabo VPS
├── Dokploy (manages SSL via Let's Encrypt)
├── Docker network: pitwall_net
│   ├── django_app      (port 8000, ASGI via Uvicorn)
│   ├── celery_worker   (default queue; no exposed port)
│   ├── celery_beat     (no exposed port)
│   ├── postgres        (port 5432, internal only)
│   ├── redis           (port 6379, internal only)
│   └── nginx_proxy     (ports 80/443 → django + static/media)
```

### 13.2 Docker Compose (Production)

```yaml
# docker-compose.prod.yml
services:

  postgres:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: pitwall
      POSTGRES_USER: pitwall
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pitwall"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks: [pitwall_net]
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks: [pitwall_net]
    restart: unless-stopped

  django:
    build:
      context: .
      dockerfile: Dockerfile.prod
    # Multi-worker is SSE-safe: each worker runs its own fan-out hub (§7.4).
    # Dev compose uses `--workers 1 --reload` — uvicorn reload is single-worker only.
    command: uvicorn pitwall.asgi:application --host 0.0.0.0 --port 8000 --workers 4
    environment:
      - DATABASE_URL=postgresql://pitwall:${DB_PASSWORD}@postgres:5432/pitwall
      - REDIS_URL=redis://redis:6379/0
      - DJANGO_SETTINGS_MODULE=pitwall.settings.production
      - SECRET_KEY=${SECRET_KEY}
    volumes:
      - django_static:/app/staticfiles
      - django_media:/app/media
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request as u; u.urlopen('http://localhost:8000/healthz', timeout=3)"]
      interval: 30s
      timeout: 5s
      retries: 3
    depends_on: [postgres, redis]
    networks: [pitwall_net]
    restart: unless-stopped

  celery_worker:
    build:
      context: .
      dockerfile: Dockerfile.prod
    command: celery -A pitwall worker -l info -c 4 -Q default,ingest
    environment:
      - DATABASE_URL=postgresql://pitwall:${DB_PASSWORD}@postgres:5432/pitwall
      - REDIS_URL=redis://redis:6379/0
      - DJANGO_SETTINGS_MODULE=pitwall.settings.production
    volumes:
      - django_media:/app/media
    depends_on: [postgres, redis]
    networks: [pitwall_net]
    restart: unless-stopped

  celery_beat:
    build:
      context: .
      dockerfile: Dockerfile.prod
    command: celery -A pitwall beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler
    environment:
      - DATABASE_URL=postgresql://pitwall:${DB_PASSWORD}@postgres:5432/pitwall
      - REDIS_URL=redis://redis:6379/0
      - DJANGO_SETTINGS_MODULE=pitwall.settings.production
    depends_on: [postgres, redis]
    networks: [pitwall_net]
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/pitwall.conf:/etc/nginx/conf.d/default.conf
      - ./ssl:/etc/nginx/ssl
      - django_static:/app/staticfiles:ro
      - django_media:/app/media:ro
    depends_on: [django]
    networks: [pitwall_net]
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  django_static:
  django_media:

networks:
  pitwall_net:
    driver: bridge
```

(Phase 3 adds a `celery_render` service: same image, `celery -A pitwall worker -l info -c 1 -Q render`, with `cpus: "1.0"` and `mem_limit: 1g` so FFmpeg renders never starve live polling.)

### 13.3 Nginx Configuration

```nginx
# nginx/pitwall.conf
upstream django_asgi {
    server django:8000;
}

server {
    listen 443 ssl;
    server_name pitwall.com www.pitwall.com;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    # SSE — long-lived streams, no buffering
    location /sse/ {
        proxy_pass http://django_asgi;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 86400;
    }

    # Static / Media served directly by nginx
    location /static/ {
        alias /app/staticfiles/;
        expires 30d;
    }
    location /media/ {
        alias /app/media/;
        expires 7d;
    }

    # Everything else → Django
    location / {
        proxy_pass http://django_asgi;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name pitwall.com www.pitwall.com;
    return 301 https://$host$request_uri;
}
```

Two companion requirements:

- **Django must trust the proxy** (production settings): `SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')` and `USE_X_FORWARDED_HOST = True` — otherwise `request.is_secure()` is `False` behind nginx, breaking allauth redirects and CSRF.
- **SSE capacity:** each viewer holds one open connection. For the 500-concurrent load target, raise `worker_connections` to 4096 in the main `nginx.conf` `events` block.

### 13.4 CI/CD with Dokploy

1. Push to `main` branch on GitHub
2. Dokploy webhook fires → pulls latest code
3. `docker-compose.prod.yml` runs `docker build` for changed services
4. Rolling restart of django + workers
5. Post-deploy hook (service name, not container name — compose-generated names vary):
   ```bash
   docker compose exec django python manage.py migrate --no-input
   docker compose exec django python manage.py collectstatic --no-input
   ```

### 13.5 Backup Strategy

```
- PostgreSQL: pg_dump daily at 02:00 UTC → stored in /backups/ on VPS
- Retention: 7 daily + 4 weekly
- Media (generated cards): retained 30 days, auto-purged by Celery task
- Redis: snapshots every 60s (default), data is ephemeral-safe (cache only)
- Known limitation: media is served straight off the VPS by nginx — no CDN /
  object storage in v1; revisit if post-race download spikes saturate bandwidth
```

### 13.6 Environment Variables (.env.example)

```env
# Django
SECRET_KEY=
DEBUG=False
ALLOWED_HOSTS=pitwall.com,www.pitwall.com
DJANGO_SETTINGS_MODULE=pitwall.settings.production

# Database
DB_PASSWORD=
# NOTE: .env files do NOT expand ${…} — never reference other vars here.
# In Docker, compose builds DATABASE_URL from DB_PASSWORD (see
# docker-compose.prod.yml). This literal URL is for local dev outside Docker:
DATABASE_URL=postgresql://pitwall:pitwall@localhost:5432/pitwall

# Redis
# Compose overrides this with redis://redis:6379/0 in-network; this literal
# URL is for local dev outside Docker:
REDIS_URL=redis://localhost:6379/0

# SMTP
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASSWORD=

# Social Auth (Google only at launch)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Web Push (VAPID)
VAPID_PRIVATE_KEY=
VAPID_PUBLIC_KEY=
VAPID_EMAIL=hello@pitwall.com

# External APIs
OPENF1_BASE_URL=https://api.openf1.org/v1
JOLPICA_BASE_URL=https://api.jolpi.ca/ergast
```

---

## 14. Feature Roadmap & Milestones

### Phase 1 — F1 Foundation (Weeks 1–6)

| # | Task |
|---|---|
| 1.1 | Project setup, Docker Compose local dev environment |
| 1.2 | Django + PostgreSQL + Redis + Celery wired up; models + migrations |
| 1.3 | F1 data ingestion (OpenF1 + Jolpica): drivers, teams, standings, calendar; **backfill 2020→current** |
| 1.4 | Base templates + Bootstrap 5 + HTMX/Alpine scaffold; dark/light mode |
| 1.5 | Homepage + F1 series hub + standings pages |
| 1.6 | Calendar + results list pages |
| 1.7 | F1 round pages: race result, qualifying, pit stops, sector times |
| 1.8 | Live polling dispatcher + SSE endpoint + `simulate_live` replay simulator |
| 1.9 | Live timing page (HTMX SSE fragments) |
| 1.10 | Driver profile pages |
| 1.11 | Deployment docs — `docs/DEPLOY.md` (Dokploy, domain, SSL); actual VPS provisioning deferred to launch |

**Milestone:** PitWall is live with F1 real-time coverage

---

### Phase 2 — Accounts & Notifications (Weeks 7–10)

| # | Task |
|---|---|
| 2.1 | User auth (email/password + Google via allauth) |
| 2.2 | User dashboard (favourites, preferences, theme persistence) |
| 2.3 | Email notification system (session reminders, race results) |
| 2.4 | Web Push notification system + VAPID setup + service worker |
| 2.5 | Notification preferences UI |
| 2.6 | Personalised feed |

**Milestone:** User accounts + notifications fully functional

---

### Phase 3 — Sharing & Content (Weeks 11–14)

| # | Task |
|---|---|
| 3.1 | Auto race card generation (Pillow) post-race |
| 3.2 | Share card UI (preview + download + WhatsApp + IG guide) |
| 3.3 | Custom card builder UI |
| 3.4 | TikTok/Reels video assembly (FFmpeg, dedicated render queue) |
| 3.5 | Shareable public profile |

**Milestone:** Sharing fully functional

---

### Phase 4 — Polish & Hardening (Weeks 15–18)

| # | Task |
|---|---|
| 4.1 | Admin dashboard + broadcast views (built on Django admin) |
| 4.2 | Championship scenario explorer ("What if") |
| 4.3 | Head-to-head driver comparison |
| 4.4 | Lap time charts + visualisations (Chart.js) |
| 4.5 | PWA manifest + service worker (offline support) |
| 4.6 | Performance audit, Lighthouse optimisation |
| 4.7 | GDPR compliance (consent banner, data deletion) |
| 4.8 | Load testing: SSE connection count under race conditions |

**Milestone:** Production-grade, feature-complete v1.0 (F1)

---

### Phase 5 — Series Expansion (Post-Launch, one at a time)

Each series ships as an independent increment: ingest module → series pages → series-specific stats. Suggested order (data reliability first):

| # | Series | Notes |
|---|---|---|
| 5.1 | F2 + F3 | FIA PDFs + Jolpica; shares F1 round weekends |
| 5.2 | NASCAR | Official stats API; stage points + playoff pages |
| 5.3 | MotoGP | Q1/Q2 split, sprint vs GP points tables |
| 5.4 | Formula E | Near-live API |
| 5.5 | WEC | Class-based standings |
| 5.6 | WRC | Stage-by-stage model |
| 5.7 | GT / GTWC | Post-session results |

---

### Phase 6 — Future (Post-Expansion)

- Public REST API (DRF) + React Native mobile app (see §15)
- Fantasy / prediction game
- Community comments / reactions on race results
- Monetisation exploration (sponsorships, premium tier)
- Instagram Graph API direct posting (for Business accounts)
- Multi-language support (ES, FR, DE, PT)

---

## 15. Future: Public API & Mobile App

### 15.1 API Strategy

V1 deliberately ships **no public JSON API** — templates consume the ORM directly. When the mobile app phase starts, a DRF layer is added over the same models (the v1.0 blueprint's `/api/v1/…` design remains the reference spec). Business logic lives in model/service functions shared by views and serializers, so the API is additive, not a rewrite.

### 15.2 Mobile Architecture (Design Now, Build Later)

```
pitwall/
├── apps/api/        # DRF (added in this phase): serializers, viewsets, JWT auth
└── mobile/          # React Native (Expo)
    ├── app/         # Expo Router file-based routing
    ├── components/
    └── lib/         # SSE/polling client
```

### 15.3 Key RN Libraries (For Future Reference)

```json
{
  "expo": "latest",
  "expo-router": "latest",
  "expo-notifications": "latest",
  "@react-native-google-signin/google-signin": "latest",
  "react-native-share": "latest",
  "@shopify/flash-list": "latest"
}
```

### 15.4 Native Advantages vs Web

| Feature | Web | React Native App |
|---|---|---|
| Push Notifications | Web Push (browser) | Native APNs / FCM |
| Background refresh | Service Worker | Background task API |
| Share to IG/TikTok | Download + manual | Native share sheet |
| Biometric auth | WebAuthn | FaceID / Fingerprint |
| Offline viewing | Service Worker cache | SQLite local cache |
| Haptic feedback | ❌ | ✅ |

The backend already stores web push subscriptions; mobile push adds a device-token table and a `POST /api/v1/me/push/device/` endpoint in the API phase.

---

## Appendix A — Series Colour Palette

| Series | Primary Colour | Hex |
|---|---|---|
| Formula 1 | F1 Red | `#E8002D` |
| Formula 2 | FIA Blue | `#1B3A73` |
| Formula 3 | Teal | `#009B77` |
| Formula E | Neon Green | `#00D2BE` |
| WEC | Cyan / ACO | `#00B4D8` |
| WRC | Rally Yellow | `#FFD000` |
| NASCAR | Dark Blue | `#003087` |
| MotoGP | Orange | `#FF6B00` |
| GT/GTWC | Silver | `#8B8B8B` |

---

## Appendix B — Data Ingestion Task Schedule (Celery Beat)

```python
# pitwall/celery.py (CELERY_BEAT_SCHEDULE excerpt)
CELERY_BEAT_SCHEDULE = {
    # Dispatcher: checks session states, raises/lowers the live-poll flag.
    # The 5s fast-poll loop self-reschedules ONLY while a session is LIVE.
    'live-dispatcher': {
        'task': 'apps.ingest.tasks.dispatcher.tick',
        'schedule': crontab(minute='*/1'),
    },
    # Standings / calendar refresh (every 15 min; cheap, cache-invalidating)
    'f1-standings-refresh': {
        'task': 'apps.ingest.tasks.f1.refresh_standings',
        'schedule': crontab(minute='*/15'),
    },
    # Session reminders (every minute check)
    'session-reminders': {
        'task': 'apps.notifications.tasks.send_session_reminders',
        'schedule': crontab(minute='*/1'),
    },
    # Nightly card cleanup
    'purge-old-cards': {
        'task': 'apps.sharing.tasks.purge_old_cards',
        'schedule': crontab(hour=3, minute=0),
    },
}
```

Post-launch series add their own beat entries + dispatcher rules (e.g., WRC stage poll every 2 min during a rally).

---

*PitWall Blueprint v1.5 — Ready to build. Start with Phase 1. 🏁*
