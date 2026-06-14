# REFERENCE: Terrain Compiler

The pipeline that discovers, scores, and compiles every **individually-campable site in the United States** into a 12-layer land-designation hierarchy, assigns each to its EPA ecoregion (Level III → Level IV), and renders it on the ecoregion explorers. A curated, personally-scored "where can I actually camp" atlas — the data backbone of the **Ecotone** app target.

Lives in `superorganism/dashboard/prototypes/`. As of last build: **5,223 campgrounds across 50 states, 555 fives.**

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
- `_build/verify_us_explorer.mjs` — headless runtime verify. Loads `ecoregion-explorer-us.html` in real Chrome (`puppeteer-core` from `servicegrid/sg-studio/node_modules` + system Chrome), asserts every spot renders as a marker with zero JS errors, and reports per-layer counts + marquee. Serve `prototypes/` with `python3 -m http.server 8899` first. Run it before every commit - build-green is not enough. (Ignores the browser's automatic `favicon.ico` 404.)

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
| 4 | BLM | ✅ | 473 | 12 Western states; dispersed-dominant (209 dispersed) |
| 5 | Tribal | ✅ | 39 | 16 states; consent-filtered curated set |
| 6 | NWR | ✅ | 53 / 22 states | Default-DENY sweep; campable refuges are rare boat-in/island needles |
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

### Layer 4 — BLM ✅ (473 rows, 12 states — verified)
The biggest national gap. The Bureau of Land Management runs ~245M acres, almost entirely in the West. BLM camping is **dispersed-dominant** — even more than national forests. The NF workflow machinery (roster → author → verify, per state, Sonnet, cap-and-salvage) **ports directly**; the conventions adapt as follows:

- **Scope: ~12 Western states, not 42** — NV/UT/CA/ID/OR/WY/AZ/NM/CO/MT (+ AK/WA). The East/West gradient collapses; it's all-West, all-dispersed-dominant. A much smaller run (likely 1–2 session legs).
- **No clean "unit" like a national forest.** BLM is organized state → district → field office, not named units. Per-state enumeration targets: **developed campgrounds** (Recreation.gov) + the **National Conservation Lands** crown jewels (National Monuments — Grand Staircase, Bears Ears, Vermilion Cliffs; National Conservation Areas — Red Rock, King Range; Wildernesses / WSAs) + the **famous named dispersed areas** + **LTVAs** (BLM-specific Long-Term Visitor Areas — Quartzsite et al., the snowbird dispersed zones).
- **Source ladder swaps** (forestcamping.com is FS-only): Recreation.gov → blm.gov field-office recreation pages → BLM National Conservation Lands → **Campendium / iOverlander / FreeRoam / The Dyrt** for the named dispersed spots (Alabama Hills, Valley of the Gods, Moab/Fruita BLM) — because BLM does **not** publish dispersed lists; that knowledge lives in the camper community (the "unofficial spots worth including") → Wikipedia (monuments/coords).
- **Type vocab:** same set **minus `shelter`** (no AT/PCT-style shelter system on BLM); `lookout` rare. Flag **LTVA** as a dispersed flavor in the row.
- **Lenses lean dark-sky hard** (many BLM monuments are certified International Dark Sky), plus Moab/Fruita MTB and desert-river paddle/boat-in.
- **Quality: run the verify pass INLINE** (BLM is small enough for 1–2 legs) so it lands verified from the start, not as author-only drafts — and hold dispersed sourcing to well-documented standouts, not every marginal pullout. (Lesson from the NF run: ~57% of the salvaged forest rows are unverified; for small layers, don't defer verify.)
- **Estimated scale:** ~600–1,000 curated rows.

Currently only co-managed wildernesses bleed into the NF layer (Kanab Creek, Yuki, La Madre); standalone BLM is entirely absent.

### Layer 5 — Tribal ✅ (39 rows, 16 states — verified)
**More different from NF/BLM than they are from each other — almost none of the dispersed playbook ports.** This is a *different artifact*: a curated, permit-gated, consent-filtered shortlist, NOT a fan-out. Key considerations:

- **Sovereignty = ~574 separate jurisdictions, not one agency.** No central enumeration, no Recreation.gov — each nation runs its own permit system, fees, seasons, rules. The "source ladder" is per-nation tribal sites/offices, wildly variable and often offline.
- **The dispersed model INVERTS** (and that was the core of NF/BLM). Casual/dispersed camping by non-members is generally **prohibited**; you camp only at designated, explicitly-permitted sites, often with a mandatory permit and sometimes a **required tribal guide** (Havasupai lottery permits, Navajo backcountry permits, guide-only Monument Valley / Antelope Canyon zones). The "pin the named dispersed" engine turns OFF.
- **A cultural-consent filter the federal layers never needed.** List a place ONLY where the nation actively invites visitors to camp; record protocols (permit-required, no-photography, no off-trail, sacred-site / member-only closures, seasonal ceremony closures). The bar is "the tribe welcomes it," not "technically possible."
- **Curation, not enumeration → the workflow shape changes.** The universe of visitor-campable tribal land is a small discrete set (~few dozen: Havasupai, Monument Valley's The View, Antelope Point on Lake Powell, tribal parks, pueblo campgrounds, White Mountain / San Carlos Apache lands). Build it as a **single curated research pass with deep per-tribe permit verification**, NOT a per-state roster→author→verify fan-out.
- **Two scoping traps:** (1) **co-management dedup** — Canyon de Chelly is an **NPS** monument, Bears Ears is **BLM/FS**; the tribal layer is for tribally-OWNED-and-run camping, don't double-list federal units; (2) **Alaska Native corporation (ANCSA) lands** are a separate category (corporations, not reservations) — **include only where a visitor can genuinely camp** (apply the campability rule; most ANCSA land isn't set up for visitor camping), not a blanket sweep.
- **Volatility:** permit systems/fees/access change often and aren't centrally tracked — entries need "verify current status" and go stale faster than federal layers.

Lenses still apply where they fit (Havasupai = a marquee 10-mi hike-in), but cultural/scenic iconicity is the real selector. **Build after BLM finishes.**

### Layer 6 — NWR ✅ (53 rows, 22 states — verified)
**The first regional layer taken national, and the cleanest application of the Default-DENY sweep.** The US Fish & Wildlife mandate is wildlife, not recreation, so ~90% of the ~570 refuges are strictly day-use - the *inverse* of the state-park layer (assume EXCLUDE; include only on an authoritative FWS statement). The campable universe is a small, scattered ~53.

- **Shape: ~7 regional sweep agents covering all 50 states, NOT a 48-state fan-out.** For a small default-DENY layer the per-state fan-out is overkill (most states return 0-1); regional sweeps (AK+HI / Desert SW+CA / Pacific NW+N Rockies / Great Plains / Midwest / South / Northeast), each enumerating only the refuges with verified camping, is faster and **verify-inline** (no salvage, no unverified tail). ~6-8 agents is the right granularity for a default-DENY layer of this size.
- **The manager-verification trap - "near a refuge is not on a refuge."** Most apparent refuge camping is actually adjacent BLM / NPS / state / Corps land. Verified excludes: Lake Umbagog (NH State Parks), the Colorado River shoreline at Imperial/Havasu (BLM), Salt Plains "campground" (Great Salt Plains State Park), Kirwin & Flint Hills (Army Corps), Chincoteague (NPS Assateague), Seedskadee & Browns-Park-dispersed (BLM). Only camping the **refuge itself** authorizes on refuge land counts.
- **The secondary-source trap.** Wikipedia/blogs routinely claim camping the official FWS regulation PDFs *prohibit* (Cache River, Big Lake, Noxubee). Every include rests on an fws.gov / recreation.gov statement; default-deny when the regs are silent.
- **Co-management absorption (the hierarchy at work).** A refuge campground co-located with a higher-layer unit resolves UP: James Kipp Recreation Area - a CMR-NWR-boundary campground the BLM actually runs - merged into the BLM / Upper Missouri River Breaks NM row at identical coords. Correct, not a bug. `audit_national.py`'s NP+NWR invariant is now **absorption-aware**: an NWR-file row missing from the curated NWR layer is OK iff it relocated to a higher layer at matching coords; a true vanish still FAILs.
- **The needles are boat-lens gold.** The campable refuges skew boat-in / paddle / sandbar / island - exactly the owner's lens: Cape Romain Bulls Island (SC), Upper Mississippi River sandbars (4 districts, MN/WI/IA/IL), Roanoke River paddle platforms (NC), Dale Bumpers White River sandbars (AR), CMR Missouri Breaks floats (MT), Kenai Swan Lake canoe routes (AK, a 5), Maine Coastal Islands / Halifax (ME).
- **Distribution:** AK 13 (Kenai/Tetlin road camps + river floats), the Colorado/Missouri/Mississippi/White river corridors, a few Western dispersed refuges (Kofa, Sheldon, Cabeza Prieta), Wichita Mountains (OK, Doris + Charons Garden). The Northeast is nearly empty (1 row). HI = 0 (all day-use/closed). Score spread: one new 5 (Kenai canoe routes) + Okefenokee's existing 5, the rest 3-4.
- **Files:** Okefenokee (the original seed) stays in `nps_nwr_blm.json`; the 52 new rows live in a dedicated `layers/nwr.json` (the blm/tribal per-layer-file pattern). Registered `nwr` in the `build_layers` load list + `integrate_staging`. Runtime-verified via the new `_build/verify_us_explorer.mjs` (5,223 markers render, NWR in the legend at 53, zero JS errors).

### The discovery toolkit — seven reusable approaches

The five completed layers produced a toolkit of distinct discovery methods. Each remaining layer inherits one or two — naming them turns the plan for 6–12 into a mapping exercise, not a fresh design each time.

1. **Roster-filter** (state parks) — enumerate every unit in an official system, keep the campable ones, one "Best camp" pick each.
2. **System-per-mode** (national forest) — a unit is a multi-mode camping *system*; enumerate the modes (developed/dispersed/backcountry/shelter/lookout); the East/West gradient sets what dominates.
3. **Spine-anchor** (BLM) — one dominant provider or institution carries the whole layer (BLM dispersed + LTVAs; later, USACE for lakes).
4. **Consent-gate** (tribal) — a per-unit policy/welcome filter; include only where camping is genuinely invited.
5. **Default-DENY sweep** (the no-camping park removals) — assume no camping, hunt the rare needles.
6. **Curated-marquee** (the "5s are rare" discipline) — destination-grade only, deliberately not exhaustive.
7. **Verify-inline vs deferred-verify** (size-dependent QA) — small layers verify as authored; large layers author then run a verify-only pass.

### Layers 7–12 — Regional layers (🟡 GA/AL/SC only) — planned national approach

Built during the original three-state pass; each needs national expansion. **NWR (layer 6) is now complete (see above)** and validated the method: map each layer to a toolkit technique, size the agent fan-out to the layer, execute. The planned approach per remaining layer:

**7 · WMA — *System-per-mode, but the "system" is the STATE, not the unit.*** This is the national forest of the East/South/Midwest and the **largest remaining layer**. Each state's WMA system has its own rule — designated hunt-camps only / dispersed-allowed / day-use-only — so determine the *state policy first*, then enumerate accordingly. The unique gate is **license + season** ("campable" = usable by a solo traveler outside hunt season or with a cheap permit). Needs the fan-out workflow + cap-and-salvage; **deferred-verify**. Highest yield, lowest verifiability.

**8 · State Forest — *Roster-filter OR System-per-mode, chosen per-state by MODEL.*** Two models (the dual model already flagged for this layer): **dispersed-throughout** (PA ~2.2M ac free primitive w/ permit, NY Forest Preserve, WA/OR DNR — treat like a mini-national-forest: note the policy, pin named primitive areas + lean-tos) vs **developed-campground** (most Southern state forests — a plain roster). MI/MN are roster-rich (~130 rustic MI state-forest campgrounds); the **shelter** type returns (Adirondack/Catskill lean-tos). **Determine the model per state before enumerating** — it is NOT "dispersed land" wholesale. Moderate yield, mostly verifiable.

**9 · Land Trust — *Default-DENY sweep, tiny.*** Most trust/preserve land is day-use; the Nature Conservancy generally bans camping. The few campable ones are special-access systems: Maine's AMC huts + Maine Island Trail (boat-in), some Western conservancies, Forever Wild AL tracts, Lula Lake. ~20–40 nationally. **Verify-inline.** The hut/island-trail systems are the marquee.

**10 · Other Parks — *Curated-marquee only — do NOT enumerate every county.*** Pin the destination-grade regional/county systems that rival state parks: Maricopa County desert parks (AZ — genuine 4–5s), East Bay Regional (CA), San Diego County, the big metro districts. Quality-gated, **not** coverage-gated — the "5s are rare" rubric applied at the *system* level. Judgment-heavy; **verify-inline**.

**11 · Scenic River — *System-per-mode where the unit is a RIVER SEGMENT and the mode is boat-in/gravel-bar.*** The owner's boat lens makes this high-value: Buffalo (AR), Current/Jacks Fork (MO), Middle Fork Salmon + Selway (ID, lottery), Green/Yampa (permit), Allagash (ME). Unique gate is **permit/lottery** for the Western multi-day floats. **Design fork:** most famous floats run *through* national forest/BLM, so the hierarchy (scenic-river = rank 11) would absorb them into NF/BLM and erase the river identity — either exempt scenic-river from downward dedup (treat it as a corridor overlay) or pin the bookable put-in/take-out. Lean: pin access points + permit regime, let the corridor be the "why."

**12 · Lake — *Spine-anchor on USACE — the BLM of this layer.*** The Army Corps of Engineers is the **#1 federal camping provider after the Forest Service** (~2,500 reservoir recreation areas), all on Recreation.gov and fully verifiable. Enumerate Corps lakes with campgrounds first, then TVA / Bureau of Reclamation / state lake rec areas; lake stays the bottom catch-all for residual reservoir rows. Highest yield-per-effort and verifiability of anything remaining — **the easy big win**.

**Execution note:** at the owner's direction we are running the regional layers in **numerical order (6 → 12)**, not the yield-optimized order below. NWR (6) ✅ done; **WMA (7) is next**.

**Suggested sequence** (yield × verifiability × effort, recorded for reference): **(1)** Lake/USACE — biggest verifiable win, Recreation.gov does the work; **(2)** State Forest — substantial, settles the dual-model question; **(3)** Land Trust — quick high-filter sweep; **(4)** Scenic River — boat-lens gold, decide the dedup fork first; **(5)** Other Parks — curated judgment pass; **(6)** WMA — its own fan-out campaign, save for last.

**Separate axis (not in these seven):** the per-spot **modular overlays** — nearby bike trails, paddle/kayak routes, hiking trails, destination restaurants — are enrichment attached to each camp, not designation layers. A distinct track for when the 12-layer hierarchy is filled out.

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
TOTAL 5,223 campgrounds · 50 states · 555 fives
score 5:555  4:1690  3:2091  2:887

national park   491  (39 states)   ✅
state park     1894  (50 states)   ✅
national forest 2214 (42 states)   ✅
blm             473  (12 states)   ✅
tribal           39  (16 states)   ✅
nwr              53  (22 states)   ✅
wma              13  (GA/AL)       🟡
state forest      3  (GA/AL)       🟡
land trust        3  (GA/AL)       🟡
other parks       6  (GA/AL)       🟡
scenic-river      9  (GA/AL/SC)    🟡
lake             25  (GA/AL)       🟡
```

**Status:** the 5 big public-land layers + **NWR (layer 6)** are **complete** (NWR national expansion 2026-06-14: 53 rows, 22 states, verify-inline). **Next:** continuing the regional layers in **numerical order** — **WMA (7)** is next (the largest remaining layer; needs the fan-out workflow + cap-and-salvage), then State Forest, Land Trust, Other Parks, Scenic River, Lake. The per-layer approach for each is documented in §6; optional later, a verify-only pass on the 26 author-only NF states. See [memory: national-forest camping discovery] for the reusable per-layer method.
