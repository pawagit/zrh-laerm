# Phase 0 — Discovery findings (2026-09-14)

Results of the remaining Phase 0 items from `01_zrh-nightflights-claude-code-prompt-v2.md`.
No application code was written. Raw samples are in `docs/samples/`, the AIP extract in
`docs/reference/`. Items that change the v2 prompt are marked **[changes v2]**.

## Summary

1. **[changes v2] adsb.lol cannot be polled every 3 s.** At 3 s cadence 35 of 40 requests
   returned HTTP 429; at 10 s cadence 12 of 12 succeeded; at 5 s the limiter triggered again.
   Sustainable: about one request per 10 s with backoff on 429.
2. **[changes v2] adsb.fi is a viable primary feed.** 200 requests at 3 s cadence, zero
   errors, same aircraft set as adsb.lol within 5 NM. Terms: non-commercial, 1 request/s,
   attribution required. airplanes.live refuses API access without an e-mail request.
3. **OpenSky token flow works** (30-minute token, 1 credit per bounding-box call, history up to
   1 hour back, 403 beyond).
4. **Runway geometry obtained from the AIP** (AD 2.12/2.13, valid from 22 JUN 2017) via an
   Internet Archive copy of a former public eAIP mirror; verified against OurAirports and
   against real tracks (cross-track error ≤ 10 m). The current eAIP is a paid skybriefing product.
5. **[changes v2] Lärmbulletin URLs are predictable**; the listing page is not fetchable with
   curl, so the fetcher probes URLs by pattern. Permit lists are far larger than "≈ 5/month":
   April 6, May 15, June 64, July 36 entries.
6. **[changes v2] The airport widgets serve 1000 days, not 10** (`amountOfDays=1000` gives
   daily counts back to 2023-10-16). July 2026 widget arrival counts per runway and bin
   equal the bulletin's daily tables exactly.
7. **[changes v2] The ADS-B on-ground flag flips at about 100 kt, not at the wheels.** Runway
   times must be derived from height above runway, with the flag as a secondary signal.
8. **[changes v2] The "Art. 5 defines Nachtzeit" citation is wrong.** The 22:00–06:00 night
   appears in the main reglement Art. 5 (noise surcharges) and Anhang 1 Art. 11; Anhang 1
   Art. 12–16 are as described in v2.

## 1. adsb.lol polling test

Endpoint `GET https://api.adsb.lol/v2/point/47.4647/8.5492/20`, identifying User-Agent.

| Test | Cadence | Result |
|---|---|---|
| 40 polls | 3 s | 5 × 200, 35 × 429 (successes at 0, 3, 6 s, then one every ≈ 30 s) |
| 12 polls | 10 s | 12 × 200 |
| 10-min run, adaptive | 5 s start | 429 at poll 4 and 5, then 32 of 34 OK at 10–20 s |

The 429 body is a plain nginx page (`docs/samples/adsblol_429_response.html`), no
`Retry-After` header. Successful responses: ≈ 12.9 KB, 24–31 aircraft within 20 NM, of
which 12–16 on the ground. Fields include `alt_baro` (feet or `"ground"`), `alt_geom`,
`gs`, `track`, `baro_rate`, `nav_qnh`, `category`, `type` (`adsb_icao`, `adsb_icao_nt`,
`mlat`, `mode_s`), `seen_pos`, `nic`, `rc`, `r`, `t`, `dbFlags`.

## 2. Alternative live feeds

| Feed | Endpoint | Result at 3 s | Terms |
|---|---|---|---|
| adsb.fi | `https://opendata.adsb.fi/api/v2/lat/47.4647/lon/8.5492/dist/20` (a v3 path exists too) | 200/200 × 200 over 10 min, ≈ 13.1 KB/response, same aircraft set as adsb.lol | personal/non-commercial, 1 request/s, must cite adsb.fi with link, 4xx/429 responses count against the limit |
| airplanes.live | `https://api.airplanes.live/v2/point/...` | 403: "Please contact us at contact@airplanes.live ... description of the project" | access by e-mail request |

adsb.fi peculiarities: top-level keys are `now` (seconds, not milliseconds as in adsb.lol),
`aircraft` (not `ac`), `resultCount`. Otherwise readsb-compatible fields.

Cross-source coverage in the 10-minute run, aircraft within 5 NM: adsb.lol 14, adsb.fi 16,
14 in common (the two extra were seen only by adsb.fi).

## 3. OpenSky token flow

- Token endpoint as in v2, `grant_type=client_credentials`, HTTP 200 in ≈ 250 ms,
  `expires_in` 1800 s, realm role `OPENSKY_API_DEFAULT`.
- `GET /api/states/all?lamin=47.1147&lomin=8.0492&lamax=47.8147&lomax=9.0492&extended=1`:
  46 states (34 on ground), ≈ 5.8 KB, header `x-rate-limit-remaining` decreased by exactly 1
  per call (3999 → 3997).
- `time=` parameter: 10 min and 50 min back both returned data (server snaps `time` by −2 s);
  4000 s back → 403 "Historical data more than 1 hour ago can only be retrieved with
  /states/own". So a collector restart can backfill up to 1 hour at 1 credit per snapshot.
- The `time` field of consecutive calls is not monotonic (1789381468, 1789381466, 1789381489):
  store OpenSky positions keyed by their own `time`, not by request time.

## 4. Runway geometry (AIP)

Source: AIP Switzerland LSZH AD 2.12 and AD 2.13, valid from 22 JUN 2017, from the Internet
Archive copy of `aip.engadin-airport.ch` (mirror itself is offline; skybriefing's eAIP is a paid
product; BAZL says the AIP "can be ordered from Skyguide"). Full extract:
`docs/reference/lszh_ad2_aip_2017-06-22_extract.txt`.

| RWY | TRUE BRG | Dimensions | THR coordinates (WGS-84) | THR decimal | THR ELEV | TORA / LDA (m) |
|---|---|---|---|---|---|---|
| 10 | 096° | 2500 × 60 | 47 27 32.18N 008 32 14.93E | 47.458939, 8.537481 | 1391 ft | 2500 / 2500 |
| 28 | 276° | 2500 × 60 | 47 27 23.76N 008 34 13.63E | 47.456600, 8.570453 | 1416 ft | 2500 / 2500 |
| 14 | 137° | 3300 × 60 | 47 28 55.53N 008 32 09.87E | 47.482092, 8.536075 | 1402 ft | 3300 / 3150 |
| 32 | 317° | 3300 × 60 | 47 27 40.65N 008 33 52.06E | 47.461292, 8.564461 | 1402 ft | 3300 / 3300 |
| 16 | 155° | 3700 × 60 | 47 28 32.57N 008 32 09.37E | 47.475714, 8.535936 | 1390 ft | 3700 / 3700 |
| 34 | 335° | 3700 × 60 | 47 26 57.39N 008 33 14.91E | 47.449275, 8.554142 | 1388 ft | 3700 / 3230 |

Geoid undulation (GUND) at all thresholds: 47.2–47.3 m = 155 ft. ARP (opennav, from AIP):
47° 27' 52.92" N, 8° 32' 57.01" E, elevation 1416 ft. RWY 28 has a 160 m EMAS at its end.

Checks: threshold-to-threshold distances 2492 / 3147 / 3243 m match the AIP lengths minus the
displaced thresholds of RWY 14 (150 m) and RWY 34 (470 m); computed bearings 96.0°, 137.3°,
155.0° match the AIP; OurAirports `runways.csv` gives the same points to 5 decimals
(its RWY 14/34 rows are runway ends, not thresholds). Real tracks: a landing on 14 and a
take-off on 28 lie within 10 m of the centreline; the exit onto a taxiway shows as 86 m.

Caveat: the 2017 data predates the planned extensions of RWY 28 and RWY 32 (approved 2024,
not built as of today). Re-verify against the current AIP when construction starts; the
coordinates should carry `source: "AIP CH AD 2.12 eff. 2017-06-22 (archive copy)"` in `rules.ts`.

## 5. Lärmbulletin: URLs, availability, structure

URL pattern (no `rev` parameter needed; `vs=1&sc_lang=de&sc_site=dxp-portal` suffices):

```
https://media.flughafen-zuerich.ch/-/jssmedia/airport/portal/dokumente/das-unternehmen/politics-and-responsibility/noise-and-sound-insulation/YYMM_lrmbulletin_<monat>.pdf?vs=1&sc_lang=de&sc_site=dxp-portal
```

- Month tokens: `januar februar mrz april mai juni juli august september oktober november dezember`
  (März is `mrz`; `maerz` and `marz` are 404).
- `lrmbulletin` naming from 2408 onward (all months 2408–2607 return 200). Older files use
  `laermbulletin` (2101, 2201, 2308, 2401, 2406, 2407 return 200; 2108, 2208, 2403 are 404
  under both spellings, so pre-2408 naming is not fully regular).
- 2608 (August 2026) is not yet published. `Last-Modified`: April → 17 Jun, May → 16 Jun,
  June → 16 Jul, July → 14 Aug, i.e. publication around the 14th–17th of the following month.
- The listing page (`/unternehmen/laerm-politik-und-umwelt/laermmonitoring/laermbulletin`,
  with or without `/de/`, browser User-Agent, `Accept-Language: de`) redirects curl to
  `/de/unternehmen`; the Sitecore layout API returns 400. The fetcher should not depend on the
  listing page: probe the pattern with HEAD from the 12th of the following month, store ETag
  and Last-Modified.

Structure (July 2026, 29 PDF pages): p1 title, p2 TOC, p3 route overview, p4–p12 one page per
runway (departures 10, 16, 28, 32, 34; arrivals 14, 16, 28, 34) with daily rows, a monthly
row, a year-to-date total, the prior-year month and its total; p13 "Flugbewegungen während
Nachtflugsperrzeit" (half-hour bins × LV/CV/NLV/NGV × Landung/Start, plus a row
"Gem. VIL Art. 39 / 39a") followed by the per-flight permit list; p14–p28 noise measurement
stations; p29 imprint.

- Departure pages have bins `00-06, 06-22, 22-23, 23-24` per route (A/B/C/D for RWY 10 etc.).
  Arrival pages have bins `00-06, 06-07, 07-09, 09-20, 20-21, 21-22, 22-23, 23-24`.
- `pdftotext -layout` scrambles the daily tables (the Total column drifts onto separate lines)
  and the half-hour summary table; the per-flight permit list parses cleanly line by line:
  `DD.MM.YYYY  HH:MM  S|L  Piste  Typ  [Verkehrsart]  Ausnahmegrund`. Verkehrsart is
  sometimes blank (2 of 36 rows in July). The daily tables need a coordinate-based parser
  (pdfjs text items with x/y), or can be skipped for arrivals because the widget (section 6)
  carries identical numbers.
- Permit list sizes: April 6, May 15, June 64, July 36. Reasons seen: schwierige
  Wetterverhältnisse, Wetterverhältnisse am Herkunftsort, betriebliche Ursache, technische
  Störung, Ambulanzflug, Staatsflugzeug, ausländisches Staatsflugzeug, Katastrophenhilfsflug,
  Messflug. Times run through the whole night (e.g. 03:18, 05:08, 05:29 ambulance flights), and
  entries at 00:00–00:58 on 01.07 belong to the night of 30.06, listed under the calendar day.
- Half-hour bins include `24:00 - 00:30`, `00:30 - 01:00`, `01:00 - 05:00`, `05:00 - 05:30`,
  `05:30 - 06:00`: the airport reports permits for the whole 22:00–06:00 night.

## 6. Airport widgets: 1000 days of daily counts

`GetArrivals?amountOfDays=N` and `GetDepartures?amountOfDays=N` accept N up to at least 1000
(oldest day returned: 2023-10-16; 100 → 2026-06-06, 400 → 2025-08-10). Dates are
`YYYY-MM-DDT00:00:00Z` calendar days. No day in the 1000-day range has a zero total.

Cross-check July 2026, arrivals 22:00–24:00 per runway: widget P14 11, P28 799, P34 181;
bulletin `Jul 26` rows (22-23 + 23-24): P14 0 + 11, P28 676 + 123 = 799, P34 133 + 48 = 181.
Also 00–06: P14 6 in both. Exact agreement. Departures in the widget have no time bins, so
the 22–23 / 23–24 departure bins still come only from the bulletin.

Consequence: the widget gives a machine-readable daily arrival series (with the 22–00 bin)
from 2023 onward, so the validation job can compare our counts against it every day without
waiting for the bulletin, and the bulletin parser can focus on departure bins and permits.

## 7. Betriebsreglement citation check

Text extracted with `pdftotext -layout` from `docs/00_betriebsreglement_flughafen_zuerich_2026.pdf`.

- Anhang 1 Art. 1 (open 06:00–23:30), Art. 12 (plan until 23:00, delays until 23:30,
  permits after 23:30; version of 10 March 2022), Art. 13 (charter starts planned until 22:00,
  delays until 22:30), Art. 14 (non-commercial: not permitted during the Nachtzeit), Art. 15
  (federal-law exemptions), Art. 16 (publication): confirmed as in v2.
- **The Nachtzeit is not defined in "Art. 5" of Anhang 1** (Anhang 1 Art. 5 is about
  access control; the main reglement's Art. 5 is about fees). The 22:00–06:00 period appears
  in: main reglement Art. 5 ("Starts und Landungen während der Nacht (22.00 bis 06.00 Uhr)",
  noise surcharges), Anhang 1 Art. 11 ("in der Zeit von 22.00 – 06.00 Uhr", noise index for
  night departures), Anhang 1 Art. 37 (idle power settings). Anhang 1 Art. 14 uses
  "Nachtzeit" without defining it; the federal definition is in VIL Art. 39 (not in the PDF,
  to be cited from the SR 748.131.1 text). `rules.ts` should cite Anhang 1 Art. 11 and VIL
  Art. 39 for the 22:00 boundary, not "Art. 5".
- Additional rule found: Anhang 1 Art. 11 bans night departures (22:00–06:00) of aircraft
  exceeding the VIL Art. 39a noise indices, and of aircraft producing more than 95 dB(A) at
  Oberglatt on northern departures. Not a time rule; noted for completeness.

## 8. Observed data characteristics that affect detection

Six transitions were captured at 3 s cadence (adsb.fi), five at 10–20 s (adsb.lol), all on
RWY 28 (take-offs) and RWY 14 (landings), QNH 1026 hPa.

- **On-ground flag ≈ 100 kt threshold.** Take-offs: last `"ground"` sample at 93–97 kt,
  first airborne-format sample at 98–111 kt with `alt_geom` still at ground level (1575 ft)
  and `baro_rate` ≤ 0. Landings: last airborne sample at 104–105 kt, first `"ground"` at
  95–99 kt. The transponder's air/ground logic switches near 100 kt, i.e. ≈ 10–15 s before
  rotation and ≈ 10–15 s after touchdown. The flag is therefore a consistency signal only.
- **`alt_baro` is pressure altitude.** At QNH 1026 an aircraft on the runway (1416 ft) reports
  ≈ 1000–1075 ft. Height above runway must use `alt_geom − 155 ft (GUND) − THR elevation`
  (observed on-ground `alt_geom` 1550–1600 ft) or `alt_baro + (nav_qnh − 1013.25) × 27 ft`.
  `nav_qnh` is present on airliners; general aviation often lacks it.
- **Surface vehicles** appear with `category` C1/C2 and `type` `adsb_icao_nt` (hex 4b5dxx,
  callsigns like TE23, ALT777, ATL008); 60–450 such records per run. Filter them out.
- **Stale positions.** Parked aircraft keep appearing with `seen_pos` up to 60 s; positions
  must be de-duplicated by (hex, `now − seen_pos`). adsb.fi at 3 s: 75 % of returned rows
  are new positions; adsb.lol at 20 s: 91 %.
- **MLAT / Mode S** rows exist (`type: mlat`, `mode_s`) with coarser positions; keep with
  lower confidence.
- Ground coverage at ZRH is good in both feeds (taxiing aircraft visible with gs 0–33 kt).

## 9. Storage estimate

Measured: ≈ 13 KB per response, 24 aircraft, 18 new positions per adsb.fi poll (≈ 10 within
5 NM), 22 per adsb.lol poll, 46 states per OpenSky bbox call (the bbox is larger than the
20 NM circle). Midday traffic; night values will be lower.

| Stream | Cadence | Polls/day | New rows/day |
|---|---|---|---|
| adsb.fi 21:30–01:30 | 3 s | 4 800 | ≈ 86 000 |
| adsb.fi rest of day | 30 s | 2 400 | ≈ 43 000 |
| adsb.lol 21:30–01:30 | 10 s | 1 440 | ≈ 32 000 |
| adsb.lol rest of day | 60 s | 1 200 | ≈ 26 000 |
| OpenSky 21:30–01:30 | 10 s | 1 440 | ≈ 66 000 |
| **Total** | | ≈ 11 300 | **≈ 250 000** |

At ≈ 150 B per row including a (hex, ts) index and daily partitions: **≈ 35–40 MB/day,
≈ 13 GB/year** with all five streams; ≈ 25 MB/day without daytime adsb.lol and OpenSky
(v2 estimated 20–30 MB/day). Raw JSON responses: ≈ 150 MB/day uncompressed, ≈ 15–20 MB/day
gzipped. Recommendation: parsed rows for everything, gzipped raw responses only for
21:30–01:30 (re-parse capability where it matters), both kept indefinitely.

## 10. Proposed detection thresholds (for `rules.ts` / detection config)

1. **Aircraft filter**: drop `category` C1/C2 and `type` `adsb_icao_nt`; keep `mlat`/`mode_s`
   rows with `position_quality = low`.
2. **Height above runway (HAR)**: `alt_geom − 155 ft − THR_elev` if `alt_geom` present, else
   `alt_baro + (nav_qnh − 1013.25) × 27 ft − THR_elev` using the aircraft's `nav_qnh` or the
   most recent `nav_qnh` seen from any aircraft within 10 min. On-runway level: HAR < 75 ft.
3. **Runway association**: cross-track ≤ 100 m from the extended centreline, along-track from
   −500 m before THR to runway length + 500 m, track within ±12° of the runway true bearing
   (direction selects 10 vs 28 etc.). Observed on-runway cross-track: ≤ 10 m.
4. **Take-off time** (wheels-off): last sample on the runway with HAR < 75 ft and gs ≥ 60 kt,
   followed by the first sample with HAR ≥ 75 ft or `baro_rate` ≥ +300 ft/min; time =
   midpoint. Confidence high if the two samples are ≤ 6 s apart, medium ≤ 30 s, low otherwise.
5. **Landing time** (wheels-on): last aligned sample on final with HAR ≥ 75 ft and the first
   with HAR < 75 ft (or `"ground"`); time = interpolation to HAR 0 using `baro_rate`
   (fallback midpoint). Same confidence bands.
6. **Fallbacks without low-altitude samples**: take-off = first aligned airborne sample within
   3 NM of the centreline with HAR < 2000 ft and gs > 90 kt, time = t − HAR / max(|baro_rate|,
   1500 ft/min), confidence low; landing = last aligned sample within 2 NM of THR with
   HAR < 1000 ft, gs 100–180 kt, time = t + (distance to THR + 300 m) / gs, confidence low.
7. **On-ground flag**: used only to confirm (a take-off needs a `"ground"` sample before,
   a landing one after, when ground coverage exists); a contradiction lowers confidence.
8. **Idempotency / dedup**: one movement per (hex, type) within 120 s; re-runs replace the
   estimate only if confidence is not lower. Go-around: a landing candidate followed by a
   climb to HAR > 300 ft without an on-runway sample → `go_around`, not a landing.
9. **Helicopters** (category A7, e.g. Rega): no runway; movement = first/last sample with
   HAR < 75 ft inside the airport polygon, confidence medium.
10. **Coverage gap**: no successful poll from any source for > 60 s inside 21:30–01:30, or
    a transition whose bracketing samples are > 60 s apart.

## 11. Proposed cadence (revised)

| Window (local) | adsb.fi | adsb.lol | OpenSky |
|---|---|---|---|
| 21:30–01:30 | 3 s (primary) | 10 s, ×2 backoff on 429 up to 60 s | 10 s (1440 credits) |
| 01:30–06:00 | 10 s | 60 s | – |
| 06:00–21:30 | 30 s | 60 s | – |

Rationale: adsb.fi resolves transitions to ±2 s; adsb.lol and OpenSky provide independence
and gap cover; the 01:30–06:00 window matters only for permit flights (ambulance, state),
which the bulletin lists. Each client uses a fixed identifying User-Agent and counts its own
429s, because adsb.fi counts rejected requests against the limit.

## 12. Open questions (one batch)

1. **Feed choice and terms**: adsb.fi as primary is non-commercial-only with mandatory
   attribution ("cite adsb.fi and include a link"). OK for this project and a later read-only
   public site? Should I also send the airplanes.live access e-mail as a third source?
2. **Cadence**: accept the revised table in section 11, or a different split?
3. **Runway geometry source**: accept the 2017 AIP data (archive copy) with the caveat noted,
   or do you have skybriefing access to download the current LSZH AD 2 pages into `docs/`?
4. **Night definition citation**: cite Anhang 1 Art. 11 and VIL Art. 39 for 22:00–06:00 in
   `rules.ts` (v2 said "Art. 5")?
5. **Validation targets**: with the widget as a daily source, propose: arrivals per day ×
   runway × bin (00–06, 06–22, 22–00) exact match ≥ 95 % of cells and total within ±2 %;
   bulletin departures 22–23 and 23–24 per runway exact match ≥ 95 %; every permit flight
   found within ±60 s. Keep these numbers?
6. **Bulletin daily tables**: parse them with a coordinate-based PDF parser (extra effort), or
   use the widget for arrivals and parse only the departure hour bins and the permit list?
7. **Storage**: keep all parsed positions from all streams (≈ 13 GB/year), or drop daytime
   adsb.lol/OpenSky (≈ 9 GB/year)? Store gzipped raw JSON for the night window only?
8. **Day attribution of permits**: the bulletin lists 00:00–05:59 permits under the calendar
   day; confirm that comparisons use calendar day (as in v2) while dashboards use the
   operational night.
9. **Untracked files**: `docs/01_...v2.md` and `docs/laermbulletin/*.pdf` (4.3 MB) are
   untracked; should they be committed?

## Appendix: files added

- `docs/reference/lszh_ad2_aip_2017-06-22_extract.txt` — AIP AD 2.12/2.13 text with source URL.
- `docs/samples/fzag_GetArrivals_1000d_2026-09-14.json`, `fzag_GetDepartures_1000d_2026-09-14.json`
  — widget responses, 1000 days.
- `docs/samples/adsbfi_v2_lat_lon_dist20_2026-09-14.json`, `adsblol_v2_point_20nm_2026-09-14.json`,
  `opensky_states_all_bbox_2026-09-14.json`, `adsblol_429_response.html` — one raw response each.

## 13. Decisions (2026-09-14, after review)

1. adsb.fi is the primary live feed (non-commercial use, attribution "Data: adsb.fi" with link
   in the UI and README); adsb.lol and OpenSky are secondary. airplanes.live: not requested yet.
2. Cadence as in section 11.
3. Runway geometry: AIP AD 2.12 valid 22 JUN 2017 (archive copy), with the caveat in section 4;
   the current LSZH AD 2 pages are not available for now.
4. The 22:00–06:00 night boundary is cited as Betriebsreglement Anhang 1 Art. 11 and
   VIL Art. 39 in `rules.ts`, not "Art. 5".
5. Validation targets kept: arrivals per day × runway × bin exact in ≥ 95 % of cells, daily totals
   within ±2 %; bulletin departure bins 22–23 and 23–24 per runway exact in ≥ 95 %; every permit
   flight found within ±60 s on the published runway with the published type.
6. Arrival counts for validation come from the airport widget; the bulletin parser covers the
   departure pages (coordinate-based parsing of the daily tables) and the permit list. Bulletin
   arrival pages are parsed only as a monthly cross-check of the widget.
7. Storage: parsed positions from all streams are kept indefinitely in daily partitions;
   gzipped raw responses are kept only for the 21:30–01:30 window.
8. Comparisons with airport data use the calendar day; dashboards use the operational night
   (a movement before 06:00 belongs to the previous evening).
9. The v2 prompt and the bulletin PDFs are committed to the repository.
10. Local environment notes for Phase 1: `pnpm` is not installed, corepack 0.34.5 is (use
    `corepack enable` and a `packageManager` field); Docker 29.5.3 / Compose v5.1.4 run; the
    local PostgreSQL 18.1 already listens on port 5432, so Docker Compose maps `db` to host
    port 5433.
