# REFERENCE: Terrain Compiler

The pipeline that discovers, scores, and compiles every **individually-campable site in the United States** into a 12-layer land-designation hierarchy, assigns each to its EPA ecoregion (Level III → Level IV), and renders it on the ecoregion explorers. A curated, personally-scored "where can I actually camp" atlas — the data backbone of the **Ecotone** app target.

Lives in `superorganism/dashboard/prototypes/`. As of last build: **4,659 campgrounds across 50 states, 515 fives.**

---

## 1. Pipeline architecture

```
data/camping/layers/*.json   ──┐
  (per-layer authored rows)    │
data/camping/curated.json ─────┤   (also the residual base: lake/USACE rows that live nowhere else)
  staging/{final,nps,nf_final} ┘
              │
              ▼   _build/build_layers.mjs        compile: hierarchy dedup + EPA ecoregion assignment
        data/camping/curated.json  ◄── the single source of truth (every compiled row)
              │
              ▼   _build/build-explorer.mjs       bake
        ecoregion-explorer.html  (GA inline)
        ecoregion-explorer-svg.html  (GA inline, offline)
        data/us/spots.json  ◄── the US explorer fetches this at runtime
        ecoregion-explorer-us.html  (all 50 states, lazy-loads L4 geometry)
```

Supporting scripts:
- `_build/integrate_staging.mjs` — folds workflow staging output (`staging/nf_final/`, etc.) into the layer files, compiles, auto-nudges ecoregion gaps. **SP/NPS staging re-processing is disabled** (it re-added rows removed during earlier audits — only national-forest staging still flows through).
- `_build/audit_national.py` — the quality gate. Run from `data/camping/`. Checks schema, type/score/layer vocab, duplicates, state-bbox containment, ecoregion-null accounting, layer↔curated invariants, marquee-unit spot-checks.
- Verify recipe (headless): `puppeteer-core` (from `servicegrid/sg-studio/node_modules`) + system Chrome on `http://localhost:8899/ecoregion-explorer-us.html`. Serve `prototypes/` with `python3 -m http.server 8899`.

**`curated.json` is both output AND input.** The compiler loads it as the base layer (it holds residual rows — lakes, USACE, scenic-rivers — that exist in no layer file), then appends the layer files and dedupes. Consequence and the single most important gotcha: **editing a row in a layer file does NOT propagate if a stale copy is already in `curated.json`** — same-rank dedup only fills empty fields, so the stale value wins. **To modify existing rows, drop them from `curated.json` first so they re-resolve fresh from the layer file**, then recompile.

---

## 2. Entry schema

One JSON object per site. Exactly these keys (16 + optional flags):

```json
{
  "name": "Cooper Creek Campground",
  "state": "GA",                     // 2-letter; the state the SITE physically sits in (units span borders)
  "county": "Union",
  "waterbody": "Cooper Creek (wild trout)",
  "category": "National Forest",     // the designation
  "landowner": "USFS",
  "operator": "Chattahoochee-Oconee NF, Blue Ridge RD (Recreation.gov)",
  "reservation": "Recreation.gov (reservable)",   // the REAL system; for dispersed, the policy
  "type": "tent",                    // see vocabulary below
  "sites": 14,                       // int count, or null where meaningless (dispersed/backcountry/shelter/lookout)
  "lat": 34.7647, "lng": -84.0735,   // AT the site, inside the named state
  "notes": "... Best camp: <the specific pick>; one night. Off FS-236 near Suches.",
  "layer": "national forest",        // one of the 12 (see hierarchy)
  "score": 5,                        // 2–5, national calibration
  "why": "One sentence, confident reviewer voice."
}
```

Conventions: `notes` ends with `Best camp: <specific site/loop/area>; one night` (or `two nights` — RARE, only when a second day unlocks a permit activity / big trail / paddle circuit / dark sky / island), then a short locator. American English, `" - "` not em-dashes. Optional `"unverified": true` flag marks author-only rows (see Quality tiers). Day-use/group-only placeholders carry `"sites": 0` and `type` `day-use`/`group-only`.

---

## 3. Type vocabulary

The fixed set (encodes *how* you sleep, not just "campground"):

`developed` · `tent` · `walk-in` · `primitive` · `dispersed` · `backcountry` · `boat-in` · `equestrian` · `shelter` · `lookout` (+ markers `group-only`, `day-use`).

Validated in both `audit_national.py` (`TYPES`) and `integrate_staging.mjs` (`VALID_TYPE`) — keep them in sync.

---

## 4. The campability hard rule

**Every entry must offer individual camping a solo traveler can actually book or use.** Exclude: day-use parks, group-only camping, picnic areas, trailheads with no camping, lodging-only (cabins/lodge with no campsites), and anything closed or transferred out of its system. Backcountry-by-permit, dispersed, boat-in, paddle-in, bike-in, shelters, and rentable lookouts all **count**.

This rule is why "Chickamauga Battlefield" (group-only) was removed, why GA's Smithgall Woods / Panola Mountain and SC's day-use parks were dropped from the state-park layer, and why every roster is reconciled against the official system before authoring.

---

## 5. Scoring rubric

**2–5, calibrated nationally — 5s are rare.**
- **5** — genuine destination worth routing a trip around (a campground under marquee peaks, a clear alpine lake, a famous dispersed basin, an island, a one-of-a-kind geology).
- **4** — strong / distinct setting.
- **3** — solid regional pick, OR promoted by the owner's lenses.
- **2** — ordinary but campable.

**The owner's lenses (promote on these):** travels with a **boat** and a **bike**, loves **hiking**. Promote sites with in-unit singletrack, marked paddle/water trails, boat-in/paddle-in/bike-in camping, marquee trail access (AT/PCT/CDT, named wilderness), or fire lookouts. Demote ordinary roadside RV reservoir loops to 2. `two nights` and second-spot picks are reserved for genuine dual-mode destinations.

---

## 6. The 12-layer designation hierarchy

The layers are an ownership/designation taxonomy, top-to-bottom by precedence (`HIER` in `build_layers.mjs`). On a geographic overlap the **higher layer wins** the merged row. Markers below: ✅ national · 🟡 partial · ⬜ empty.

| # | Layer | Status | Rows | Notes |
|---|---|---|---|---|
| 1 | National Park | ✅ | 491 / 39 states | Every campable NPS unit, per-campground |
| 2 | State Park | ✅ | 1,894 / 50 states | All 50 state-park systems |
| 3 | National Forest | ✅ | 2,214 / 42 states | All forests; 8 states have no NF land |
| 4 | BLM | ⬜ | 0 | The big Western gap — next priority |
| 5 | Tribal | ⬜ | 0 | Small, permit-gated, curated |
| 6 | NWR | 🟡 | 1 / GA | Okefenokee only |
| 7 | WMA | 🟡 | 13 / GA, AL | |
| 8 | State Forest | 🟡 | 3 / GA, AL | |
| 9 | Land Trust | 🟡 | 3 / GA, AL | |
| 10 | Other Parks | 🟡 | 6 / GA, AL | County/municipal/authority |
| 11 | Scenic River | 🟡 | 9 / GA, AL, SC | |
| 12 | Lake | 🟡 | 25 / GA, AL | USACE & utility reservoirs (residual base) |

### Layer 1 — National Park ✅
**Granularity: one row per developed campground**, plus one aggregate row per unit for a real dispersed/backcountry/wilderness permit program (e.g. "Yosemite NP - Wilderness (backcountry)"). Yellowstone = ~12 rows; Acadia = 4; a backcountry-only unit = 1.
**Discovery:** enumerate every NPS unit nationally that offers individual camping (nps.gov find-a-park camping pages); exclude no-camping and group-only units. Include national parks, seashores, lakeshores, recreation areas, preserves, monuments, historical parks, **parkways** (Blue Ridge, Natchez Trace) and scenic rivers with NPS-run camping. Per-campground rows verified against nps.gov campground pages + Recreation.gov; coords from nps.gov maps / Wikipedia.
**Field conventions:** `category` = the unit designation; `landowner` = `National Park Service (<full unit name>)`; row name = `<Unit short name> - <Campground name>`; state = the state THAT campground sits in.
Most NPS units in any given state are historic sites / battlefields / day-use with no camping — the campability filter does heavy work here.

### Layer 2 — State Park ✅
**Granularity: one row per park** (mixed-camping parks use the developed count in `sites`).
**Discovery:** the official state-parks agency roster reconciled against the Wikipedia "List of <State> state parks" — this catches **system drift**: renames (GA Gordonia-Alatamaha → Jack Hill), transfers out (GA Standing Boy Creek → Columbus; AL Florala), closures, and brand-new parks. "State park system" means the parks agency's units (State Parks, Recreation/Resort Areas, campable State Natural Areas) — **NOT** state forests, WMAs, or historic sites unless a historic site has real individual camping. Per-state quirks: NY = OPRHP parks only (NOT DEC forest-preserve campgrounds); California includes campable State Beaches/SRAs.
GA / AL / SC were hand-authored first as the reference standard. The other 47 ran through the fan-out workflow (roster → author → verify).

### Layer 3 — National Forest ✅
**The distinct method: a forest is a multi-MODE camping SYSTEM, not a campground roster.** Enumerate **per forest** (each state has 1–18 forests); for each forest capture **all** modes as `type`s:
- **developed** — the enumerable spine.
- **dispersed** — *"pin the named, note the rest"*: you cannot enumerate dispersed (legal across most of a forest), so pin the named/notable dispersed areas, road corridors, and designated-dispersed zones, AND record the forest's general dispersed **policy** in the row (free / MVUM-governed / 14-day). Never map every pullout.
- **backcountry** — wilderness areas (hike-in/boat-in).
- **shelter** — trail shelters where a National Scenic Trail crosses (AT/PCT/CDT/NCT).
- **lookout** — rentable fire lookouts & FS cabins (deliberately first-classed — "sleeping in the forest at a unique spot").

**East/West gradient** sets what dominates: Eastern forests are developed-campground-heavy; Western forests are dispersed-and-lookout-dominant (capture the famous dispersed basins/roads + lookouts; the developed list is secondary).
**Source ladder:** forestcamping.com (cleanest per-forest tables WITH site counts) → Recreation.gov gateways/facility pages → fs.usda.gov camping-cabins pages → Wikipedia (coords + wilderness acreage).
**GA built first as the worked template** (developed + 11 AT shelters + named dispersed/wilderness corridors). Result type spread: developed 723, backcountry 434, dispersed 298, primitive 271, **lookout 182**, shelter 117, tent 67, equestrian 56, walk-in 33, boat-in 33.

### Layer 4 — BLM ⬜ (next — adaptations locked)
The biggest national gap. The Bureau of Land Management runs ~245M acres, almost entirely in the West. BLM camping is **dispersed-dominant** — even more than national forests. The NF workflow machinery (roster → author → verify, per state, Sonnet, cap-and-salvage) **ports directly**; the conventions adapt as follows:

- **Scope: ~12 Western states, not 42** — NV/UT/CA/ID/OR/WY/AZ/NM/CO/MT (+ AK/WA). The East/West gradient collapses; it's all-West, all-dispersed-dominant. A much smaller run (likely 1–2 session legs).
- **No clean "unit" like a national forest.** BLM is organized state → district → field office, not named units. Per-state enumeration targets: **developed campgrounds** (Recreation.gov) + the **National Conservation Lands** crown jewels (National Monuments — Grand Staircase, Bears Ears, Vermilion Cliffs; National Conservation Areas — Red Rock, King Range; Wildernesses / WSAs) + the **famous named dispersed areas** + **LTVAs** (BLM-specific Long-Term Visitor Areas — Quartzsite et al., the snowbird dispersed zones).
- **Source ladder swaps** (forestcamping.com is FS-only): Recreation.gov → blm.gov field-office recreation pages → BLM National Conservation Lands → **Campendium / iOverlander / FreeRoam / The Dyrt** for the named dispersed spots (Alabama Hills, Valley of the Gods, Moab/Fruita BLM) — because BLM does **not** publish dispersed lists; that knowledge lives in the camper community (the "unofficial spots worth including") → Wikipedia (monuments/coords).
- **Type vocab:** same set **minus `shelter`** (no AT/PCT-style shelter system on BLM); `lookout` rare. Flag **LTVA** as a dispersed flavor in the row.
- **Lenses lean dark-sky hard** (many BLM monuments are certified International Dark Sky), plus Moab/Fruita MTB and desert-river paddle/boat-in.
- **Quality: run the verify pass INLINE** (BLM is small enough for 1–2 legs) so it lands verified from the start, not as author-only drafts — and hold dispersed sourcing to well-documented standouts, not every marginal pullout. (Lesson from the NF run: ~57% of the salvaged forest rows are unverified; for small layers, don't defer verify.)
- **Estimated scale:** ~600–1,000 curated rows.

Currently only co-managed wildernesses bleed into the NF layer (Kanab Creek, Yuki, La Madre); standalone BLM is entirely absent.

### Layer 5 — Tribal ⬜
Small and special: a curated, permit-gated handful, not a big enumeration. Havasupai / Havasu Falls, Monument Valley / Navajo, reservation parks. Distinct permit systems per nation.

### Layers 6–12 — Regional layers (🟡 GA/AL/SC only)
Built during the original three-state pass; need national expansion.
- **NWR** (1, Okefenokee) — refuge canoe-platform / wilderness camping; mostly permit paddle systems.
- **WMA** (13) — wildlife management areas; dispersed/primitive, usually a state license/GORP-type pass required to be on the land.
- **State Forest** (3) — the **state-government analog of national forests**: same dual model (designated campground spine + dispersed where the state permits), with the dispersed share varying hugely by state. Dispersed-heavy: PA (~2.2M ac, free primitive w/ permit), NY (state forests + Forest Preserve lean-tos), MI (~130 rustic state-forest campgrounds + dispersed), MN. Designated-only: most Southern states (GA/AL). NOT "dispersed land" wholesale.
- **Land Trust** (3) — private conservation land with public camping (Forever Wild, Lula Lake).
- **Other Parks** (6) — county / municipal / authority parks.
- **Scenic River** (9) — Wild & Scenic / state scenic river corridors (gravel-bar & outfitter camps).
- **Lake** (25) — USACE & utility reservoir campgrounds; the residual catch-all that lives in `curated.json`, not a layer file.

---

## 7. Compiler dedup — `same(a, b)` in `build_layers.mjs`

Hard-won rules (in order). Two rows merge (higher layer wins) if:
1. **Okefenokee special-case** — both names contain "okefenokee" (one wilderness system, platforms spread far).
2. **Cross-state guard** — `a.state !== b.state` → **never merge** (same-named parks exist in different states: Wildcat Mountain SP in WI vs Wildcat Den in IA, etc.).
3. **Exact normalized name** within 0.25° (~28 km).
4. **Prefix match** (one name starts with the other) within 0.07°.
5. **Distinctive-word match within 0.009° (~1 km)** — share a 5+ letter word AFTER removing the **stoplist** of generic camping/geography words (`campground, recreation, creek, river, lake, falls, wilderness, mountain, canyon, ... upper/lower/north/south, island, reserve, preserve`). **Disabled entirely for NP↔NP pairs** because every campground in an NPS unit shares the unit's distinctive prefix word (Yosemite's Upper/Lower/North Pines all share "pines"+"yosemite") and would false-merge.

Why each clause exists: clause 2 was added when same-named parks in different states started merging at national scale; the stoplist in clause 5 was added when "Mulky Campground" false-merged into "Cooper Creek Campground" (shared generic "campground"); the NP-NP exemption was added when Yosemite/Kings Canyon/Zion campground clusters collapsed to one row each.

---

## 8. Ecoregion assignment

Point-in-polygon against EPA Level III/IV ecoregions (`data/us/us_l3.geojson` + lazy `data/us/l4/<L3>.json`). Each row gets `l3code/l3/l4code/l4`.
- **EPA Level IV is CONUS-only** — **AK and HI rows carry null ecoregions by design** (they still render and browse; they're just absent from the ecoregion drill). ~188 AK/HI nulls are expected.
- **Coastal/island slivers** can fall in a geometry gap. `integrate_staging.mjs` auto-nudges CONUS gap rows outward in a spiral until they land in a polygon, then drops them from `curated.json` to re-resolve. A handful of true offshore islands (Biscayne keys, Channel Islands, Dry Tortugas, Anclote, Patos) stay null — acceptable.
- **State-bbox validation** (in `audit_national.py`) catches coordinates that drift across a state line (Teton border NF sites tagged ID but coords in WY; Bighorn Canyon boat-in on the MT half; the NM-agent's Texas-panhandle Lake Marvin error). Fix = clamp into the tagged state's bbox, or re-tag, or drop if it's the wrong state entirely.

---

## 9. Orchestration — the fan-out workflow & cap-and-salvage rhythm

National layers run as a **Workflow** (`_build` has no copy; the script is authored inline). Shape, per the proven template:
```
pipeline(STATES, rosterAgent, [authorBatches(chunk 12–15) → verifyAgent])
```
- **roster** (per state) enumerates the system + all campable sites with hints. **author** batches write full rows to `staging/<layer>_final-or-nf/`. **verify** (per state) cross-checks, dedupes, spot-checks coords, assembles the final per-state file.
- **Model:** author/verify on **Sonnet** (cost); rosters benefit from the strong model but Sonnet is acceptable.
- **Cost reality:** national forests alone were ~thousands of rows; expect a run to hit the **session limit** (rolling ~5-hour window — NOT the monthly cap) after ~12–14 states.

**Cap-and-salvage rhythm (proven over 4+ legs):**
1. Run caps mid-flight → completion notification lists `<failures>`.
2. **Salvage inline (free):** states whose result shows only *verify* failures have complete authoring — assemble their `staging/.../<ST>.json` (dedupe within state, mark `unverified`), integrate, audit, commit. States with *author* failures are partial — defer.
3. **Resume** the same `runId` with `resumeFromRunId` — the journal replays completed agents for free. Trim the script's `STATES` to exclude already-salvaged states (avoids rework/conflict).
4. Repeat until the state list is exhausted.
The journal makes every leg lossless; total spend accumulates but no work repeats. (If the journal is ever unavailable, mine completed `StructuredOutput` tool calls out of the workflow transcript JSONL — that's how the 50-state park gap was recovered.)

---

## 10. Quality tiers

- **Verified** — full author + verify pass (dedup, coordinate spot-check, coverage-gap check). GA/AL/SC and the national-forest final 14 (OK,OR,PA,SC,SD,TN,TX,UT,VT,VA,WA,WV,WI,WY); all state parks and NPS.
- **Author-only (`unverified: true`)** — authored to the full template and **audit-clean** (schema-valid, deduped at compile, coords in-state, zero bbox violations) but not verify-cross-checked. Currently **1,259 NF rows across 26 states** (AK AR AZ CA CO FL ID IL IN KS KY LA ME MI MN MO MS MT NC ND NE NH NM NV NY OH). A **verify-only pass** (one agent/state, cheap) upgrades these to parity and closes the gaps the verify agents flag (e.g. OR's Diamond Lake, Wallowa Lake).

---

## 11. Operational recipes

**Add / extend a layer:**
1. Author rows into `data/camping/layers/<layer>.json` (or run the workflow → `staging/`).
2. If modifying existing rows: **drop them from `curated.json` first** (the staleness gotcha, §1).
3. `node _build/build_layers.mjs` (compile) → check the layer-vs-curated count for unexpected merges.
4. `node _build/build-explorer.mjs` (bake).
5. `cd data/camping && python3 ../../_build/audit_national.py` → must be FAIL 0.
6. Headless verify on :8899; commit with explicit pathspec.

**Integrate workflow staging:** `node _build/integrate_staging.mjs` (validates, dedupes vs existing, appends to layer files, compiles, auto-nudges ecoregion gaps). Reads `staging/nf_final/` for forests; SP/NPS dirs are intentionally skipped. Per-layer done-states differ (`NF_DONE = {GA, AL}` keeps SC because SC forests weren't in the original build).

**Persisted scripts:** `_build/` is git-ignored (holds `node_modules` + raw geometry), so the pipeline scripts are **force-added** to track them. Geometry (`data/us/*.geojson`, ~73MB) and `node_modules` stay ignored — geometry is re-downloadable.

---

## 12. Current standings (last build)

```
TOTAL 4,659 campgrounds · 50 states · 515 fives
score 5:515  4:1500  3:1789  2:855

national park   491  (39 states)   ✅
state park     1894  (50 states)   ✅
national forest 2214 (42 states)   ✅
blm               0                ⬜  next
tribal            0                ⬜
nwr               1  (GA)          🟡
wma              13  (GA/AL)       🟡
state forest      3  (GA/AL)       🟡
land trust        3  (GA/AL)       🟡
other parks       6  (GA/AL)       🟡
scenic-river      9  (GA/AL/SC)    🟡
lake             25  (GA/AL)       🟡
```

**Next:** BLM (large, dispersed-dominant West) → Tribal (small curated) → national expansion of layers 6–12 → optional verify-only pass on the 26 author-only NF states. See [memory: national-forest camping discovery] for the reusable per-layer method.
