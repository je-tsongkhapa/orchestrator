# REFERENCE — AI Vending Machines (Superorganism)

Superorganism's slice of the vending standard, by app archetype — the inward, self-facing machines. Framework = 4 archetypes (Console · Studio · Game · Sandbox) + an agent-native layer (ServiceGrid side). ServiceGrid's machines: `REFERENCE-vending-machines.md`.

## The standard — 8 traits

1. **Cheap, on-hand input** · 2. **One bounded outcome** · 3. **No per-unit incumbent for the small buyer** · 4. **Low liability** · 5. **Horizontal** (domain is a parameter) · 6. **Reasoning over messy input is the work** · 7. **High leverage** · 8. **A real moat** — a domain-specific structure (decode-table / criteria-matrix / closure-tie-out), not raw generation.

## Console (3)

**LandRead** — a credit + parcel photos -> parcel geometry/acreage + slope/access/drainage/zoning read
- *moat:* parcel-geometry computation + slope/zoning read keyed to located data · *buyer:* land buyers, small builders, homesteaders evaluating a lot

**GrowReport** — a credit + crop-symptom photos (+ crop) -> a scouting action sheet (ranked causes, severity, IPM plan)
- *moat:* a symptom->cause differential key + a crop-specific IPM action library · *buyer:* homesteaders, growers, gardeners

**TierList** — a credit + a corpus you hold + a worth-axis -> a defended tier-list (S/A/B/C/F, boundaries defended)
- *moat:* a criteria-matrix decomposing the axis into per-tier admission thresholds · *buyer:* anyone ranking their own collection (albums/films/games)

## Studio (11)

**PatchPlay** — a credit + a target sound/look (or your corpus) -> a collectible parameterized generator (patch/brush/rig)
- *moat:* a synthesis/idiom KB + parameterization + dedup-against-your-library · *buyer:* producers, illustrators, motion/3D & technical artists

**LookLift** — a credit + a reference whose character you want -> a reusable, content-agnostic treatment (grade/LUT/style)
- *moat:* a look-decode model + parameter-stability + a look-match-delta check · *buyer:* colorists, photo/video editors, designers

**TileForge** — a credit + a reference / your raw corpus -> a seamless material atom (loop/texture/icon), seam verified
- *moat:* a recurrence + seamlessness engine (zero-click/wrap/grid closure) · *buyer:* producers, game/3D/web makers, motion artists

**FrameKit** — a credit + the thing you keep starting from scratch -> a collectible fill-in creative scaffold
- *moat:* a codified convention/idiom grammar + a structural lint · *buyer:* makers who rebuild the same empty start every time

**RigWire** — a credit + a control behavior you keep rebuilding -> a re-bindable control component (macro/mod-matrix)
- *moat:* a parameterization engine over control + a transport dry-run check · *buyer:* makers re-wiring the same multi-parameter move

**ArtifactBuilder** — a credit + your own materials -> a finished personal artifact (keepsake/gift/memory-book)
- *moat:* a personal-corpus -> artifact pipeline + a coherence/finish check · *buyer:* individuals turning their materials into a finished piece

**Compression** — a credit + one long artifact -> its compression by harvesting the best parts (clip reel/tl;dr/supercut)
- *moat:* an in-item salience/compression model (load-bearing parts, faithful) · *buyer:* someone who can't justify a long thing's full runtime

**Bespoke** — a credit + a topic/corpus -> a made-for-you consumable (explainer/briefing/audiobook/supercut)
- *moat:* source-grounding + a coverage check (cited, no gaps) · *buyer:* someone who wants a topic's payload without the labor

**Sequencer** — a credit + a corpus -> a worth-bearing sequence/playlist (ordered by inter-item fit)
- *moat:* a between-items fit-verdict optimized over the whole arc · *buyer:* someone with the WHAT but not the ORDER

**TasteModel** — a credit + your revealed corpus -> an extracted, reusable taste-lens (named/weighted axes)
- *moat:* a decode-table lifting implicit preference into explicit criteria · *buyer:* a power-consumer who wants to see + steer their taste

**Transmute** — a credit + a media artifact + a target medium -> the same content faithfully re-bodied (doc<->video, podcast->article)
- *moat:* a structural-skeleton extractor + a two-way fidelity tie-out · *buyer:* anyone repurposing content across media

## Game (3)

**Co-op** — a credit + a mission/scenario -> one co-op run teaming with AI allies to a shared goal
- *moat:* a team-coordination engine + a shared-goal evaluator · *buyer:* people who want the experience of teaming with capable AI allies

**Guided Journey** — a credit + a journey setup (+ personal seeds) -> a gamemaster-run solo journey to an ending
- *moat:* a gamemaster pacing/branching engine + a real ending · *buyer:* interactive-fiction / solo-RPG players, guided-journey seekers

**Lockbox** — a credit + a setup (+ optional your material) -> a fair self-contained solvable you crack
- *moat:* a solvable-generation + fairness engine (proves a unique solution) · *buyer:* players wanting a fair designed brain-teaser

## Sandbox (7)

**TimeWarpKiln** — a credit + a seed/design -> max harvestable stock after N weeks of compressed world-time, or a design-verdict
- *moat:* a time-warp speed governor with determinism-equivalence · *buyer:* worldbuilders maturing a seed; anyone testing a design

**LegendsForge** — a credit + a founding seed + a run length -> a readable saga of a once-only history
- *moat:* append-only causal sediment + seeded-RNG · *buyer:* GMs, fiction/lore writers

**RuinsCartographer** — a credit + a world-state/seed -> a replayable chronicle of how the present came to be
- *moat:* deterministic replay + an intervention-provenance ledger · *buyer:* studios/researchers/GMs documenting a world's origin

**SocietyFork** — a credit + a society -> fork it (keep one to inhabit / compare N)
- *moat:* society-state capture + fork semantics + a divergence layer · *buyer:* strategists comparing outcomes; anyone wanting their own society

**VivariumNudge** — a credit + an intervention -> injected into a running vivarium, propagated to a world-scale consequence
- *moat:* a live-intervention engine + autonomous amplification · *buyer:* people running/watching a persistent agent world

**AgentDraft** — a credit (per agent) -> a specific agent instantiated into your world as a persistent inhabitant
- *moat:* an agent catalog + persistent-identity transfer · *buyer:* people populating their own society; collectible-agent buyers

**CapsuleForge** — a credit + a world/history-slice -> a beautiful, collectible generative story/comic, canon-coherent
- *moat:* a world-canon consistency engine + a collectible-render pipeline · *buyer:* collectors, fans, worldbuilders

*Banked 2026-06-06. Lives in the orchestrator repo so the SO dashboard ingests it.*
