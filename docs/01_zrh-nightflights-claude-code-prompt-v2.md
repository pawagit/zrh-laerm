# ZRH Night Operations Monitor — Claude Code Build Prompt (v2, revised 2026-09-14)

Revision of `00_zrh-nightflights-claude-code-prompt.md` after a fact check of the
regulation text, the data sources and the local environment. Every change against v1
is marked **[changed]** or **[new]**. Decisions were taken interactively on 2026-09-14.

## Context

Zurich Airport (LSZH / ZRH) operates under the Betriebsreglement (PDF in `docs/`).
Anhang 1 defines the night regime:

- Art. 1: the airport is open 06:00–23:30 local.
- Art. 12 (gewerbsmässiger Verkehr): commercial take-offs and landings may be
  *scheduled* until 23:00; delayed take-offs and landings are permitted without
  special approval until 23:30 ("bewilligungsfreier Verspätungsabbau"); after 23:30
  an exception permit (Ausnahmebewilligung) is required.
- **[new] Art. 13 (Charterflüge)**: charter *take-offs* may only be scheduled until
  22:00, delayed charter take-offs are permitted until 22:30, after 22:30 a permit is
  required. Landings of charter flights fall under Art. 12.
- **[new] Art. 14 (nichtgewerbsmässiger Verkehr)**: non-commercial take-offs and
  landings are not permitted during the Nachtzeit at all (permit only for unforeseen
  extraordinary events). The reglement defines night as 22:00–06:00 (Art. 5), so the
  ban starts at 22:00.
- Art. 15: exempt are flights permitted at night by federal law (VIL Art. 39d). Per
  the airport's own summary: emergency landings, ambulance flights (Rega/HEMS), police,
  disaster relief, Swiss military aircraft, and BAZL-approved state aircraft.
- Art. 16: permits issued are published — in practice in the monthly Lärmbulletin.
- **[new] Airport slot practice (not law)**: take-offs are planned until 22:45 and
  landings until 22:55. Shown as reference lines only.

The regulation uses "Starts und Landungen" — the physical take-off (wheels off) and
landing (wheels on). It does **not** use off-block / pushback time. Public flight
trackers often show off-block time as "Departed"; off-block to wheels-off at ZRH can
be 10–40 minutes, so such numbers are unusable for compliance checks.

**Goal:** an independent, self-hosted web app that derives true runway times from
ADS-B data collected live around ZRH and shows, per day/month, how many movements
actually occurred after each threshold (22:00, 22:30, 23:00, 23:30, 00:00, per rule
set), the distribution of movements in the last hour before curfew, delay causes we
can infer, and a comparison with the airport's own published figures.

**[changed] Data starts with the collector.** There is no historical backfill: the
credentials at hand cover only OpenSky's REST API (max 1 hour back), and Trino
historical access was deliberately not pursued. All numbers refer to the period from
collector start onward.

Solo project, single user initially, may later be made public (read-only).
Privacy-first, open-source-friendly, no paid APIs.

## Stack (fixed — do not substitute)

- Backend: **[changed] Node.js 24** + Express, TypeScript
- Frontend: React + Vite + Tailwind, TypeScript
- Database: **[changed] PostgreSQL 18**, no TimescaleDB (plain partitioned tables)
- Collector/jobs: TypeScript; Python allowed only for one-off exploration notebooks
- Deployment: Docker Compose locally (`db`, `api`, `web`, `ingest` services);
  Google Cloud Run in Phase 6. **[changed] The collector runs on the developer's
  machine via Docker Compose from Phase 2 onward** so data accrues while later
  phases are built; migration to Cloud Run happens in Phase 6.
- Package manager: pnpm via corepack (not installed globally yet), monorepo with
  `apps/api`, `apps/web`, `packages/shared`, `packages/ingest`
- Tests: Vitest. Lint/format: ESLint + Prettier, strict TS.

Local environment (verified): Node 24.13, PostgreSQL 18.1, Docker Compose 5.1,
pdftotext available, Python 3.14. Git repo exists with an initial commit.

## Data sources (verified 2026-09-14)

1. **[changed] adsb.lol live API — primary.**
   `GET https://api.adsb.lol/v2/point/{lat}/{lon}/{radius_nm}`, no account, ODbL 1.0.
   Per aircraft: `hex`, `flight`, `r` (registration), `t` (type), `alt_baro`
   (feet or the string `"ground"`), `gs`, `track`, `lat`, `lon`, `seen_pos`, `seen`,
   `category`. Verified: 22 aircraft within 10 NM of ZRH, 16 of them on the ground.
   Rate limits are **not documented**: single polite client, identifying User-Agent,
   exponential backoff, and a cadence that stays conservative (see defaults below).
2. **[changed] OpenSky Network REST API — secondary.** OAuth2 client-credentials
   (token endpoint `https://auth.opensky-network.org/auth/realms/opensky-network/protocol/openid-connect/token`,
   tokens valid 30 min). `GET /api/states/all` with a bounding box; standard users
   get 4,000 credits/day, one credit per small-bbox request; states at most 1 hour
   back, 5 s time resolution. Credentials come from `_credentials/opensky_credentials.json`
   (`clientId`, `clientSecret`) and go into `.env` as `OPENSKY_CLIENT_ID` /
   `OPENSKY_CLIENT_SECRET`. Not used: Trino/Impala (no access), `/flights/*` (block
   times only), `/tracks` (experimental, 30 days).
   Positions from both sources are stored with a `source` tag and merged per
   aircraft; detections record which source(s) contributed.
3. **[changed] adsb.lol daily archives — gap-fill only.** GitHub releases
   `adsblol/globe_history_YYYY` (`vYYYY.MM.DD-planes-readsb-prod-0.tar.aa/ab`),
   readsb trace JSON per ICAO hex, ODbL/CC0, ~2 GB/day. Used by an on-demand job
   (Phase 6) that re-derives movements for nights where the collector had a gap.
4. **Flughafen Zürich AG published data — validation** (all raw files/responses are
   stored alongside parsed values so parsing can be redone):
   - **[changed] 10-day widgets** on the Flugbewegungsstatistik page, JSON:
     `https://dxp-fds.flughafen-zuerich.ch/Flightmovements/GetDepartures?amountOfDays=10`
     → per day: counts per departure route × runway (`A_10` … `O_34`) and `total`;
     `…/GetArrivals?amountOfDays=10` → per day: counts per runway (P14, P16, P28,
     P34) × bins `00-06`, `06-22`, `22-00` and `total`. **No per-flight data, no
     times.** Numbers are marked "temporary" by the airport.
   - **Monthly Lärmbulletin PDF** (listing page
     `flughafen-zuerich.ch/unternehmen/laerm-politik-und-umwelt/laermmonitoring/laermbulletin`,
     files like `media.flughafen-zuerich.ch/…/2509_lrmbulletin_september.pdf`):
     (a) per-runway daily tables of departures and arrivals with hour bins that
     include `22-23 Uhr` and `23-24 Uhr` (plus monthly and prior-year rows);
     (b) "Flugbewegungen während Nachtflugsperrzeit": permit counts per half-hour
     bin × Verkehrsart (LV, CV, NLV, NGV) × Landung/Start — **this table counts
     permit flights only, not all movements after 23:00**;
     (c) per-flight permit list: Datum, Lokalzeit (minute precision), Bewegung
     (S/L), Piste, Flugzeugtyp, Verkehrsart, Ausnahmegrund. Roughly 5 flights/month.
     Parsed automatically in TypeScript with a review view (raw vs parsed, manual
     correction possible).
   - Monthly `monatliche-flugbewegungen_YYYYMM.pdf` and
     `entwicklung-der-flugbewegungen.pdf`: totals only, low priority.
5. **Reference data**: OpenSky aircraft database
   (`https://s3.opensky-network.org/data-samples/metadata/aircraftDatabase.csv`,
   `doc8643AircraftTypes.csv`) for ICAO hex → registration, type, operator; adsb.lol
   already delivers registration and type per position. Operator classification
   lists (Rega/HEMS, Swiss Air Force, police, state, cargo, charter operators) are
   curated in `rules.ts` with a source citation per entry — never invented.

## Core algorithm — runway time detection

For every aircraft observed within the collection area:

- **Take-off time** = first position where on-ground transitions true→false, OR if
  ground coverage is missing, the first airborne position within 3 NM of a runway
  centreline with baro altitude < 2,500 ft and groundspeed > 90 kt. Take the earlier
  reliable signal. Record detection method, contributing source(s) and a confidence
  score.
- **Landing time** = symmetric: last airborne position on final approach / first
  on-ground position within the airport polygon.
- **Runway** inferred from track and position at the moment of take-off/landing
  (10/28, 14/32, 16/34); runway geometry taken from the AIP, not guessed.
- **Off-block time** (if visible in ground data) — stored separately, never conflated.
- All timestamps in UTC; conversion to Europe/Zurich only in the API/presentation
  layer. Thresholds are local time; DST handled correctly.
- **Threshold semantics**: a movement is "after HH:MM" when its local runway time
  is ≥ HH:MM:00 (matches the airport, which bins a 23:30 start into 23:30–24:00).
- **[changed] Classification**: `scheduled_commercial`, `charter`,
  `business_aviation`, `cargo`, `state`, `military`, `hems_rega`, `unknown`, each
  mapped to the airport's Verkehrsart (LV, CV, NLV, NGV) for comparisons. Exempt
  categories are flagged with a reason, never deleted. Charter vs scheduled is only
  partly decidable from ADS-B; uncertain cases carry a `classification_confidence`.
- **[changed] Rule sets** applied per movement, all in `rules.ts` with article refs:
  Art. 12 (23:00 / 23:30 / 00:00, all commercial), Art. 13 (charter take-offs
  22:00 / 22:30), Art. 14 (NGV from 22:00), planning cutoffs 22:45 / 22:55 (info only).
- **[new] Permit matching**: bulletin permit entries are matched to our movements by
  date, local time (±2 min), movement type, runway and aircraft type.

## Validation (redefined)

**[changed]** There is no public per-flight runway-time source, so the ±60 s
per-movement target of v1 cannot be tested in general. Validation is:

1. **Counts**: our daily counts per runway and hour bin (22–23, 23–24, and the
   widget's 22–00) versus the Lärmbulletin tables and the 10-day widgets; daily
   totals per runway versus both. Report per day×runway cell: ours, published,
   delta, coverage. Proposed targets (confirm in Phase 0): exact match in ≥ 95 % of
   the 23–24 h cells, daily totals within ±2 %.
2. **Per-flight**: every bulletin permit flight must be found within ±60 s of the
   published minute, on the published runway, with the published type.
3. Mismatches are listed individually so causes (coverage gap, classification,
   detection) can be traced.

## Frontend — what I want to see

1. **Dashboard** for a selected month: movements after 22:00 (NGV / charter starts),
   22:30, 23:00, 23:30, 00:00; take-offs vs landings; exempt vs non-exempt; per
   Verkehrsart; comparison with the Lärmbulletin permit count for after 23:30.
2. **Daily timeline** 22:00–01:00: every movement as a dot, coloured by category;
   hover shows callsign, registration, type, operator, runway, detected time,
   off-block time (if known), source(s), confidence, matched permit (if any).
   Rule lines at 22:00, 22:30, 22:45, 22:55, 23:00, 23:30, 00:00.
3. **Histogram** of take-off and landing times in 5-minute bins, **[changed]
   21:30–00:30** so the charter thresholds are visible, aggregated over the period.
4. **Operator / route breakdown** of late movements (top 15 operators, top destinations).
5. **Trend view**: monthly counts per threshold over all collected months.
6. **Validation page**: count comparison tables and permit-flight matches, with the
   stats above; bulletin review view (raw vs parsed).
7. **Data quality page**: coverage gaps per night, per-source availability, flights
   with low confidence, unclassified aircraft.
8. **Export**: CSV of any filtered view.
Design: clean, data-dense, Swiss-neutral; dark mode; no clutter. German UI labels
with an English toggle later — for now, German only.

## Phases — deliver in this order, stop after each and wait for review

### Phase 0 — Discovery (no code yet) — partly done
Done on 2026-09-14: regulation text, OpenSky access, adsb.lol API/archives, airport
widgets, Lärmbulletin structure, local toolchain. Remaining: runway geometry from
the AIP, a short polling test against adsb.lol and the OpenSky token flow, the
bulletin listing page structure and file naming across months, storage estimate from
a real sample, concrete detection thresholds. Report findings and ask questions in
one batch.

### Phase 1 — Skeleton
Monorepo, Docker Compose, Postgres schema + migrations, empty Express API with
health endpoint, Vite React app with routing shell, README with setup steps, scripts
`pnpm lint`, `pnpm test`, `pnpm typecheck`.

### Phase 2 — Collector + detection **[changed]**
Live collector polling adsb.lol and OpenSky in the collection area: all day at low
cadence, denser from 21:30 to 01:30 local; raw positions stored with source tag;
runway time detection with unit tests on synthetic and real samples; idempotent
re-runs; logging with counts. Deployed locally via Docker Compose and left running
from this point on.

### Phase 3 — Validation
Daily fetch of the 10-day widgets; Lärmbulletin fetcher, parser and review view;
validation report as JSON and rendered on the Validation page; tune thresholds until
the targets are met or explain why not.

### Phase 4 — Operations & data quality **[changed, was Backfill]**
Gap detection per night, restart resilience, health/metrics endpoint, daily summary
job, data-quality page data, retention policy "keep everything" implemented as
partitioned tables.

### Phase 5 — Frontend
All views listed above, wired to real data. Responsive; usable on a phone.

### Phase 6 — Hardening
Auth for admin/ingest endpoints (simple token), read-only public mode flag,
Dockerfiles and deployment for Cloud Run (collector as always-on service), backup
script for Postgres, adsb.lol archive gap-fill job, `docs/methodology.md` explaining
exactly how runway times are derived so the numbers can be defended publicly.

## Working rules

- Before each phase, restate the plan in ≤ 10 bullet points, then execute.
- Never fabricate data or thresholds; if a source is unavailable, say so and propose
  alternatives.
- Every decision that affects the numbers (thresholds, exemptions, classification,
  rule sets) lives in `packages/shared/src/rules.ts` with comments referencing the
  Betriebsreglement article.
- Commit after each phase with a meaningful message; small, reviewable commits within
  phases.
- Prefer boring, well-maintained libraries. No ORMs heavier than Drizzle or Kysely.
- Everything in `.env.example`; no secrets in the repo (`_credentials/` stays
  gitignored).
- When uncertain about aviation semantics, ask rather than guess.

## Defaults proposed for Phase 0 confirmation

- Collection area: bounding box ±0.35° latitude / ±0.50° longitude around the ZRH
  reference point 47.4647 N 8.5492 E (about 20 NM); adsb.lol radius 20 NM.
- Cadence: day 30 s (adsb.lol only); night 21:30–01:30 local: adsb.lol every 3 s,
  OpenSky every 10 s (≈ 1,440 credits/night, within the 4,000/day budget).
- Storage: keep all raw positions indefinitely (estimate 20–30 MB/day, to be
  measured in Phase 2).
- Day attribution: dashboards use the operational night (a 00:20 movement belongs
  to the previous evening); comparisons with airport data use the calendar day, as
  the airport does.

Start with the remaining Phase 0 items.
