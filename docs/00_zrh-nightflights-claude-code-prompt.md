# ZRH Night Operations Monitor — Claude Code Build Prompt

## Context

Zurich Airport (LSZH / ZRH) operates under a Betriebsreglement whose Anhang 1, Art. 12 defines:

- Commercial take-offs and landings may be *scheduled* until 23:00 local.
- Delayed take-offs and landings are permitted without special approval until 23:30 local ("bewilligungsfreier Verspätungsabbau").
- After 23:30 an exception permit (Ausnahmebewilligung) is required; these are published in the airport's monthly Lärmbulletin.
- Exempt from the night rules: emergency/ambulance (Rega), police, disaster relief, Swiss military, and BAZL-approved state aircraft.
- Non-commercial traffic (Art. 14) is not permitted at night at all.

The regulation uses the terms "Starts und Landungen" — the physical take-off (wheels off) and landing (wheels on) on the runway. It does **not** use off-block / pushback time. Public flight trackers (Flightradar24 etc.) often show off-block time as "Departed", which makes their numbers unusable for checking compliance. Off-block to wheels-off at ZRH can be 10–40 minutes.

**Goal:** an independent, self-hosted web app that derives true runway times from ADS-B data and shows, per day/month, how many movements actually occurred after 23:00 and after 23:30 — plus the distribution of movements in the last hour before curfew, delay causes we can infer, and a comparison with the airport's own published figures.

This is a solo project. I am the only user initially; it may later be made public (read-only). Privacy-first, open-source-friendly, no paid APIs.

## Stack (fixed — do not substitute)

- Backend: Node.js 22 + Express, TypeScript
- Frontend: React + Vite + Tailwind, TypeScript
- Database: PostgreSQL 16 (TimescaleDB extension optional if it helps)
- Ingestion jobs: TypeScript scripts run via cron / Cloud Run Jobs; Python allowed only for a one-off data exploration notebook, not for production code
- Deployment target: Docker Compose locally; Google Cloud Run. Keep it a single `docker-compose.yml` with `db`, `api`, `web`, `ingest` services.
- Package manager: pnpm, monorepo with `apps/api`, `apps/web`, `packages/shared`, `packages/ingest`
- Tests: Vitest
- Lint/format: ESLint + Prettier, strict TS

## Data sources

1. **OpenSky Network** historical data — primary source. Free for non-commercial use, requires an account. Access via the Trino/SQL endpoint (`state_vectors_data4`, `flights_data4` tables) or the REST API for recent data. I will supply credentials via `.env`. Document exactly which tables/columns are used.
2. **adsb.lol** daily archive dumps — fallback / cross-check source, no account needed. Investigate the archive format before committing to it.
3. **Flughafen Zürich AG** published data — for validation:
   - "Flugbewegungsstatistik" page: daily take-off/landing lists for the last 10 days with runway times
   - Monthly "Lärmbulletin" PDF: count of movements after 23:30 with exception permits
   - Monthly flight movement PDFs
   Scrape/parse these where feasible; store raw files alongside parsed values so parsing can be redone.
4. Reference data: aircraft registry (ICAO hex → registration, type, operator) from OpenSky's aircraft database CSV; airline/operator classification (commercial scheduled, charter, business aviation, state, Rega/HEMS, military).

## Core algorithm — runway time detection

For every flight touching LSZH:

- **Take-off time** = first state vector where `onground` transitions true→false, OR if ground coverage is missing, the first airborne position within 3 NM of the runway centrelines with baro altitude < 2'500 ft and groundspeed > 90 kt. Take the earlier reliable signal. Record the detection method and a confidence score.
- **Landing time** = symmetric: last airborne position on final approach / first `onground = true` within the airport polygon.
- **Runway** inference from heading and position at the moment of take-off/landing (runways 10/28, 14/32, 16/34).
- **Off-block time** (if visible in ground data) — store it separately, never conflate.
- All timestamps stored in UTC; convert to Europe/Zurich only in the API/presentation layer. Curfew thresholds are in local time; DST changes must be handled correctly.
- Classify each movement: `scheduled_commercial`, `charter`, `business_aviation`, `cargo`, `state`, `military`, `hems_rega`, `unknown`. Exempt categories are flagged, not deleted.
- Build a validation harness: for the last 10 days, compare our detected runway times against the airport's published lists; report median/95th-percentile deviation and mismatches. Target: within ±60 s for ≥95 % of movements.

## Frontend — what I want to see

1. **Dashboard** for a selected month: movements after 23:00, after 23:30, after 00:00; split take-offs vs. landings; exempt vs. non-exempt; comparison to the Lärmbulletin figure for after-23:30.
2. **Daily timeline** 22:00–01:00: every movement as a dot on a time axis, coloured by category, hover shows callsign, type, operator, runway, detected time, off-block time (if known), detection confidence.
3. **Histogram** of take-off times in 5-minute bins from 22:30 to 00:00, aggregated over the selected period — this shows whether traffic bunches right before 23:00 / 23:30.
4. **Operator / route breakdown** of late movements (top 15 operators, top destinations).
5. **Trend view**: monthly counts after 23:00 / 23:30 over all ingested months.
6. **Validation page**: our data vs. airport's published data, with the deviation stats above.
7. **Data quality page**: coverage gaps, flights with low confidence, unclassified aircraft.
8. **Export**: CSV of any filtered view.
Design: clean, data-dense, Swiss-neutral; dark mode; no clutter. German UI labels with an English toggle later — for now, German only.

## Phases — deliver in this order, stop after each and wait for my review

### Phase 0 — Discovery (no code yet)
- Verify current OpenSky access methods and table schemas; verify the adsb.lol archive format; check what the airport's statistics page and Lärmbulletin actually expose. Report findings, uncertainties and any blockers. Propose the concrete detection thresholds. Ask me any questions in one batch.

### Phase 1 — Skeleton
- Monorepo, Docker Compose, Postgres schema + migrations, empty Express API with health endpoint, Vite React app with routing shell. README with setup steps. CI-style scripts: `pnpm lint`, `pnpm test`, `pnpm typecheck`.

### Phase 2 — Ingestion
- Ingest one sample day from OpenSky, store raw state vectors (or a reduced subset) and derived movements. Implement runway time detection with unit tests on synthetic and real samples. Idempotent re-runs. Logging with counts.

### Phase 3 — Validation
- Scraper/parser for the airport's 10-day lists. Validation report generated as JSON + rendered on the Validation page. Tune thresholds until the ±60 s target is met or explain why it can't be.

### Phase 4 — Backfill
- Backfill August of the current year (then extend backwards on request). Rate-limit aware, resumable, progress logging.

### Phase 5 — Frontend
- All views listed above, wired to real data. Responsive; usable on a phone.

### Phase 6 — Hardening
- Auth for admin/ingest endpoints (simple token), read-only public mode flag, Dockerfiles for Cloud Run, backup script for Postgres, `docs/methodology.md` explaining exactly how runway times are derived so the numbers can be defended publicly.

## Working rules

- Before each phase, restate the plan in ≤10 bullet points, then execute.
- Never fabricate data or thresholds; if a source is unavailable, say so and propose alternatives.
- Keep every decision that affects the numbers (thresholds, exemptions, classification rules) in a single `packages/shared/src/rules.ts` with comments referencing the Betriebsreglement article.
- Commit after each phase with a meaningful message. Small, reviewable commits within phases.
- Prefer boring, well-maintained libraries. No ORMs heavier than Drizzle or Kysely.
- Everything in `.env.example`; no secrets in the repo.
- When uncertain about aviation semantics, ask rather than guess.

Start with Phase 0.
