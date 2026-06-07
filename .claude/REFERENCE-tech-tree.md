# REFERENCE — Tech Tree (Languages)

> **Superorganism strategic doc — concept, not yet built.** The SO "tech tree" reconceived as a tree of **language-fluencies**: every domain of knowledge or wisdom is cast as a *language*, and the tree tracks fluency in each. The payoff is a growing, code-switched **shared medium between the user and the AI**. Companions: the app archetypes (`org-docs/.claude/REFERENCE-app-archetypes.md`) and the vending catalog (`REFERENCE-so-vending-machines.md`). The shipped CEFR-Spanish skill-tree is the seed / first strand. *Implementation deferred; this captures the design.*

## 1. The core idea

Build the tree around **the set of languages you want to speak.** As you climb a strand you acquire its vocabulary, and that vocabulary becomes the **shared language between you and the AI**. The endpoint isn't a fluency score — it's a personal **pseudolanguage**: a *code-switching composite* of all your languages, the private medium you and the AI actually talk in, growing node by node as you unlock it.

## 2. What "language" means here

Not spoken tongue. A **language** = *any symbolic system with a vocabulary of primitives + a grammar for combining them, that you can internalize to fluency and **speak in**.* Under that one definition, **any domain of knowledge or wisdom** is a candidate strand:

| Strand | Primitives | a "language" because… |
|---|---|---|
| **Spanish** | words + syntax | the obvious case |
| **C** | types · pointers · control-flow + idioms | you think *in* it; a program is an utterance |
| **Baduk** (Go) | stones + shapes ("the language of stones") | a game is an utterance; shapes are vocabulary |
| **dots-and-loops** | musical primitives + phrasing | a musical language |
| **Stoicism** | dichotomy of control · *prohairesis* · assent | a wisdom-language; a judgment is an utterance |
| …*etc etc* | — | every domain with a *way of speaking* qualifies |

The thing that makes this more than a polyglot flex: **each language imports its concepts.** Go hands you *aji, sente, shape*; C hands you *pointer, lifetime, allocation*; music hands you *tempo, key, resolution*. So the composite isn't mixed vocabulary — it's a **thinking-medium**, a personal *latticework of speakable lenses*. The tree isn't "languages I'm learning"; it's **the set of fluencies I want to think in.**

> The selection question — *which* languages — dissolves: the set is **open** (taste as the curriculum). The real design is the two problems below: **framing** (turning a domain into a language) and **the composite** (fusing the fluencies).

## 3. Knowledge → wisdom is the fluency axis

Low in a strand you're learning the words (**knowledge**); high in it you speak with taste and judgment (**wisdom**). CEFR A1→C2, generalized: **lexicon → grammar → idiom → native intuition.** The ladder *is* the knowledge→wisdom climb, and it is what the tree gates.

## 4. Design problem ① — Framing (turn any domain into a language)

A domain becomes a strand by filling this schema:

| Slot | What it is | Go | C | Stoicism *(wisdom)* |
|---|---|---|---|---|
| **Lexicon** | the primitives/terms you must know | stones, *atari* | types, operators | dichotomy of control, *prohairesis* |
| **Grammar** | how they legally combine | shape, joseki | idioms, control-flow | apply the dichotomy *before* you assent |
| **Idiom / wisdom** | the tacit register: proverbs, taste, the master's move | pro intuition, proverbs | the idiomatic style | the view from above; the obstacle is the way |
| **Fluency ladder** | the A1→C2 progression (what the tree gates) | read shapes → whole-board | hello-world → systems | knows the term → embodies *apatheia* |
| **Utterance** | what "speaking" is | a move | a program | a *lived response* |
| **decode ↔ encode** | recognize an instance's meaning / produce one | read a position / play | read code / write it | see "not in my control" / respond with equanimity |

This **generalizes the existing skill-tree card model** (`instance → decode`): `sentence → translation` (spoken languages) and `situation → concept/move` (concepts) are just two fillings of `decode ↔ encode`.

**Stress-test — the make-or-break case is a *wisdom* domain, where lexicon/grammar go fuzzy.** Stoicism fills the schema cleanly *and* surfaces a refinement the formal domains hid: for wisdom domains, **"speaking" is a lived judgment, and top-fluency is *embodiment*, not recitation.** So `utterance` ranges *a move* (Go) → *a program* (C) → *a lived response* (Stoicism), and the knowledge→wisdom axis is starkest exactly where the domain is fuzziest.

## 5. Design problem ② — The composite (fuse the fluencies)

The shared you↔AI medium, assembled from unlocked strands:

- **Growth** — each strand *donates* its lexicon + idioms; the composite's expressive range = the **union** of what you've unlocked.
- **Code-switch rule** — reach for whichever language has the **sharpest word** for the thought (*aji* from Go, *pointer* from C, *andante* from music).
- **The AI's part** — the AI **speaks back *in* the composite**, scaled to your fluencies. *Design choice:* stay strictly inside your unlocked set, or deliberately reach **one step past your edge** to pull you forward (comprehensible-input / *i+1*).
- **Private + co-tracked** — gibberish to anyone else; to you two, **maximally dense**, because both sides track the unlocked set.

**Example utterance:** *"this refactor is aji-keshi — we're spending the pointer before we need it; let it breathe, andante."* Every loan-word is load-bearing — Go · C · music · natural language fused in one sentence.

## 6. Structure

- A **pure tree of language-fluencies** — one node-kind. *(The "assemble-a-console-from-pages" idea that rode along in the seed note separated out as its own vending machine — **PageForge**, in `REFERENCE-so-vending-machines.md`. It is **not** part of this tree.)*
- **Within a strand:** the A1→C2 fluency ladder.
- **Across strands:** the composite (the fusion layer).

## 7. Relation to the existing skill-tree

The shipped **CEFR-Spanish** skill-tree (explore = vocab, decks from the user's own Anki, explore-only after the prescribed "exploit" was stripped) is the **seed and first strand**. The new payoff — *the AI speaking your emerging composite back to you* — **sidesteps the exact thing that killed the exploit:** a *prescribed* practice-conversation was redundant with Anki + daily work, but the AI conversation **is** the medium, not an assigned exercise. The tree gates how the AI talks to you. Organic, not homework.

## 8. Open questions (for implementation)

- **The fluency ladder per domain** — how to define A1…C2 concretely for Go, C, a wisdom domain.
- **How fluency is built / measured** — the card/practice model per strand (the existing `instance→decode` card is the seed).
- **Tree topology** — prerequisites within a strand, and whether strands cross-feed (does Go fluency feed a "strategy" strand?).
- **The composite's representation** — tracked formally (an "unlocked-vocabulary" set the AI reads) or emergent?
- **The AI's frontier policy** — strictly-inside vs *i+1*, and how aggressively.
- **Ground "dots-and-loops"** — cited here as the musical strand but the specific system isn't yet detailed.
- **Selection** — the set is open (taste); is there any gating principle, or purely desire?

## Appendix — the seed (verbatim)

The original phone scrawl, preserved as-is:

```
texh tree documentation
i should build the tech tree around what set of languabes i want to speak and let that emergent bocabulary be the shared labguage between me and the ai. the pseudolanguage will be a code switching composite of all of the various languages
dots and loops as a language
console apps as an a la carte vending machine. assemble which pages you want, get console app.
```

*(The console-apps line became PageForge; the rest is this doc.)*

Key crystallizations that followed (the user's framing, verbatim):
- *"'language' is more abstract than merely spoken language. it encompasses both C as well as 'Baduk' the language of stones and dots and loops as well etc etc"*
- *"domain of knowledge or wisdom is framed as a language to develop fluency in. the design is just in how to frame language as well as the composite"*

*Documented 2026-06-06. Concept; implementation deferred.*
