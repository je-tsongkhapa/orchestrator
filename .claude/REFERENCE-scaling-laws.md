# Scaling Laws — History and Taxonomy

> Source: harvested from a 2026-04-08 conversation thread building toward a general scaling-dimension framework. Some specific numerical claims (Kaplan ratios, Llama token-to-parameter ratios, DeepSeek-R1 token multiples) carry over from the source and should be verified against original papers before being acted on. Marked inline with `[verify]`.
> Revisit when: discussing scaling-law strategy; designing the sovereign model registry; reasoning about which capability layer to invest in next; mapping new domains to known scaling dimensions.

---

## Three eras (load-bearing periodization)

The history of scaling laws breaks into three substrate eras, each defined by where compute is most productively spent.

### Era 1 — Hardware substrate (1965–~2015)

Scaling law dynamics applied to the silicon layer itself.

- **Moore's Law** (Gordon Moore, 1965) — transistor density doubling every ~2 years. Underwrote the entire personal computing era.
- **Dennard Scaling** (Robert Dennard, 1974) — power consumption per transistor decreasing as transistors shrank, allowing frequency to scale without thermal blowup. Effectively dead by ~2005–2006: gates became too thin to suppress current leakage and voltages couldn't be reduced further.
- **Moore's Law slowdown** — commonly cited as effectively over by ~2015 in the original "doubling density at constant cost" form. Not literally dead (transistor counts still climb), but the historical doubling cadence has degraded and the cost-per-transistor curve flattened.
- **Successor strategies** (active): architectural innovation (multi-core, chiplets, heterogeneous accelerators), specialization (GPUs → TPUs → application-specific accelerators), efficiency techniques (sparsity, quantization, distillation).

### Era 2 — Pre-training scaling (2017–~2024)

The transformer architecture (Vaswani et al., 2017) made LLM pre-training the new dominant scaling axis. The era is defined by three sequential corrections to "bigger is better":

- **Kaplan et al.** (OpenAI, 2020) — first formal pre-training scaling laws. Suggested model size should scale faster than training data given a fixed compute budget. `[verify: specific ratios — the source claimed 5.5x model / 1.8x data per 10x compute increase]`
- **Chinchilla** (DeepMind / Hoffmann et al., 2022) — corrected Kaplan dramatically. Found that for compute-optimal training, model size and data tokens should scale roughly in equal proportion. Implication: most contemporary models were undertrained relative to their parameter count.
- **Beyond-Chinchilla** (~2023–2024) — empirical observation that loss continues to decrease well past Chinchilla-optimal token counts, particularly when *inference cost* is the binding constraint rather than training cost. Llama 2/3 trained with very high token-to-parameter ratios. `[verify: source claimed up to 1,875 tokens per parameter]` This solved the "Chinchilla trap": Chinchilla-optimal models are training-cheap but expensive to serve.

### Era 3 — Post-training and inference (2024–present)

From 2020 to early 2024, scaling = pre-training scaling. From 2024 onward, the field added two new axes:

- **Post-training** (RLHF, RLAIF, constitutional AI, RL with verifiable rewards) — capability gains from feedback loops on top of a pre-trained base, often dramatic and parameter-efficient.
- **Inference-time compute** — spending more compute *at generation time* via longer reasoning chains, search, best-of-N, or process reward models. DeepSeek-R1 (early 2025) demonstrated that pure RL on reasoning traces could match OpenAI's o1-class performance, generating substantially more tokens per query in exchange for much higher accuracy. `[verify: source claimed 10–100x more tokens per query]`

The Era 3 unlock is *which axis to spend the marginal compute dollar on* — pre-training, post-training, or inference. The frontier labs are now optimizing across all three simultaneously.

---

## Known scaling laws (5 categories, MECE)

The organizing principle: **where in the AI lifecycle the compute is spent.** Each category contains multiple specific laws or strategies.

### 1. Hardware substrate (compute per dollar per watt)

| Law | Status |
|---|---|
| Moore's Law (transistor density) | Largely exhausted in original form |
| Dennard Scaling (voltage/power) | Dead since ~2006 |
| Architectural specialization (GPUs, TPUs, accelerators) | Active, primary frontier |
| Efficiency scaling (sparsity, quantization, distillation) | Active |

### 2. Pre-training (static knowledge acquisition)

| Law | Description |
|---|---|
| Kaplan scaling | Parameter-dominant allocation given fixed compute |
| Chinchilla scaling | Data-parameter parity given fixed compute |
| Beyond-Chinchilla / inference-aware training | Overtrain smaller models when inference cost dominates |
| Data quality scaling | Higher-quality data → better parameter efficiency |

### 3. Post-training (behavioral refinement)

| Law | Description |
|---|---|
| RLHF / RLAIF | Human or AI feedback as reward signal |
| Constitutional / rule-based RL | Encoded principles as reward shape |
| Synthetic data pipelines | Self-play, self-distillation, reasoning-trace generation |
| RL with verifiable rewards | Reward signal from automated verification (math, code, tests) |

### 4. Inference-time compute (thinking harder per query)

| Law | Description |
|---|---|
| Chain-of-thought / extended reasoning | Longer reasoning chains per query |
| Best-of-N sampling + verification | Generate N candidates, select via verifier |
| Tree search with process reward models | Search reasoning trees, score intermediate steps |
| Parallel reasoning | Multiple independent reasoning passes, aggregate |

### 5. Agentic / environment interaction (acting in the world)

| Law | Description |
|---|---|
| Tool use scaling | More tools + better tool selection |
| Multi-step workflow execution | Longer autonomous task horizons |
| Multi-agent coordination | Specialist ensembles, orchestration |
| Self-replication / agent spawning | Parallel agent instances on subproblems |

---

## Candidate future scaling laws (NOT MECE — overlap noted)

These are speculative and have real overlap problems. Several of them are subcomponents of others viewed from a different angle. Listed as candidates worth tracking, not as a clean taxonomy.

### 6. Memory and context (persistent knowledge substrate)

- Long-context scaling (context windows)
- Persistent memory across sessions
- Retrieval-augmented scaling (external knowledge bases)
- Episodic learning (learning from interaction history)

*Overlap note:* parts of this could be a subcomponent of inference-time compute (long context = more tokens to attend over) rather than an independent law. The persistent / episodic part is more genuinely distinct because it changes the system's state, not just its working set.

**Memory depth as a quantifiable variable.** The sub-components above are usually framed as architectural choices (longer context windows, persistent stores, retrieval pipelines), but they can be reframed as a single measurable scaling variable: *memory depth* — how much context a system can maintain and update across time, broken into three measurable axes. (1) **Encoding distance** — how far back in the interaction history the relevant information was originally committed; measurable as the gap between the moment of encoding and the moment of retrieval. (2) **Retrieval accuracy** — rate of correct lookup against ground truth when the system needs information that was previously encoded; measurable as a precision/recall over a known corpus. (3) **Integration quality** — whether retrieved information actually changes the system's downstream behavior or sits inert in context; measurable by comparing decisions made with vs without the retrieved information present. All three are concrete metrics that can be tracked over time, which makes "memory depth" closer to a real scaling variable than the catch-all "memory" or "context" framings. The scaling claim: more memory depth across all three axes → predictable improvement in capability, particularly for tasks where the relevant information was encoded many steps back.

### 7. Action-feedback loop density

- Rate of environment feedback per unit time
- Richness and verifiability of reward signal from real-world outcomes
- Cycle time between action and verified result

This is the leading candidate for the next dominant scaling law if there is one. It's the substrate that agentic scaling (#5) actually depends on — agents don't compound just by spawning more, they compound by closing more verified feedback loops per unit time. Robotics is the forcing function because physical environments produce clean failure signal that language environments don't. Connects to **verifier's law** (Jason Wei): the ability to train on a task is proportional to how easily verifiable that task is.

#### Substrate bets for action-feedback loop density

Four categories of substrate investment underwrite this candidate scaling law. They are not independent — they are coupled, with simulation fidelity acting as the rate-limiting unlock for the others.

**Simulation fidelity and speed** — probably the highest-leverage substrate right now. The action-feedback loop is currently bottlenecked by how fast you can run physically realistic environments. If real-world robot experiments take minutes and have a 30% failure rate due to hardware, your loop is slow and noisy. A sim that runs 10,000x faster than real-time with accurate physics closes that gap. The unlock condition for the whole next scaling law is probably sim-to-real transfer working reliably — and that's still genuinely unsolved. Whoever solves it owns a critical piece of infrastructure.

**Failure signal clarity** — underrated. The value of robotics as a forcing function is that failure is legible — but only if you've built good instrumentation around what actually failed and why. A robot dropping a glass tells you something went wrong. It doesn't automatically tell you whether the world model was wrong, the policy was wrong, the perception was wrong, or the action execution was wrong. Systems that can decompose failure into attributable causes generate dramatically better training signal per experiment. This is more of a tooling and methodology substrate than a hardware one, but it compounds just as hard.

**Persistent memory architecture** — substrate for the dominant law rather than the law itself. The systems that will benefit most from dense action-feedback loops are the ones that can actually maintain and update state across thousands of interaction cycles rather than reconstructing it from context each time. Working on this now means you're ready when the loop density law kicks in. This is probably where the most interesting research is happening that isn't fully visible yet.

**Curation infrastructure for synthetic experience** — connects back to the scaling law loop. As action-feedback cycles generate massive volumes of interaction data, the bottleneck shifts to what's worth keeping and feeding back into pre-training. The labs that have good automated curation pipelines — that can distinguish "this interaction was genuinely novel and informative" from "this was the ten-thousandth variation of the same task" — will compound faster than those that don't. This is almost a meta-substrate: it's what makes the entire loop efficient rather than just fast.

**The rate-limiting substrate.** Of the four, simulation fidelity is the rate-limiting step on everything else. You can't build good curation infrastructure if you don't have enough diverse interaction data to curate. You can't validate persistent memory architectures if your action-feedback loop is too slow to generate the sequences they need to learn from. Sim is the substrate that unlocks the other substrates. The substrate-investment sequencing implied: simulation fidelity first, failure-signal instrumentation alongside it, persistent memory and curation infrastructure positioned to capitalize once sim is producing data fast enough to need them.

### 8. Multimodal fusion (cross-modal reasoning)

- Vision-language-audio-action unified models
- World model scaling (physics simulation, spatial reasoning)
- Embodied intelligence (robotics + perception + reasoning)

*Overlap note:* the embodied intelligence subcomponent is hard to separate from #7 (action-feedback loop density). Vision-language fusion is more genuinely distinct.

### 9. Recursive self-improvement (AI improving AI)

- AI-generated training data quality
- AI-designed architectures (neural architecture search at scale)
- AI-optimized training curricula
- AI-written code improving AI systems

*Overlap note:* substantially overlaps with the curation step of the scaling law loop (see below) and with synthetic data pipelines (#3). The distinct claim is *full pipeline automation* — not just using AI as a tool inside training, but having AI design the next training run.

### 10. Social / network intelligence (collective scaling)

- Multi-model collaboration (specialist ensembles)
- Cross-organization knowledge sharing
- Swarm intelligence patterns
- Human-AI team scaling

*Overlap note:* the multi-model and swarm subcomponents are extensions of #5 (multi-agent coordination). The cross-organization part is genuinely distinct — it's about pooled capability across separately-owned systems.

### 11. Temporal / continual learning (learning without forgetting)

- Online learning during deployment
- Curriculum-aware training (progressive difficulty)
- Forgetting-resistant architectures

*Overlap note:* the curriculum part overlaps with #3 (synthetic data pipelines selecting training order). The continual / forgetting-resistant part is genuinely distinct and currently unsolved.

### 12. Causal model depth (cross-environment reasoning generalization)

- Number of cause-and-effect steps a system can accurately trace in a *novel* environment
- Robustness of causal chains as the inference distribution diverges from the training distribution
- Generalization of causal reasoning *across* environments, not performance *within* one

**Distinct from reasoning depth within a context window.** Reasoning depth measures how deep a chain of inference can go *within a known environment* given enough context — it's a within-distribution property. Causal model depth measures how robustly the same chain generalizes *across environments* — specifically, when the system encounters a situation that wasn't in its training distribution, how many causal steps can it still trace correctly before the chain collapses. A system with high reasoning depth but low causal model depth performs well on familiar problems and falls apart on unfamiliar ones. A system with high causal model depth maintains performance across novel environments because its underlying causal model is more general than its training distribution. The distinction matters because a benchmark that only tests within-distribution reasoning depth will rank shallow-but-narrow systems highly while missing the property that determines real-world robustness.

**Quantifiable.** Track inference accuracy on causal chains in novel environments as a function of chain length, and measure the falloff curve. A system whose accuracy stays flat across chain length in unfamiliar settings has high causal model depth. A system whose accuracy collapses at chain length 2–3 in unfamiliar settings has low causal model depth even if it can reason 20 steps deep within familiar settings.

*Overlap note:* shares some surface area with #11 (continual learning) — both are about behavior outside the training set — but causal model depth is specifically about *causal* reasoning generalization, while #11 is about learning new things without forgetting old ones. Distinct mechanisms even though both address out-of-distribution behavior. Also related to #8 (multimodal fusion) via the world-model subcomponent, but a multimodal system with weak causal model depth still fails on novel reasoning tasks regardless of how well it fuses modalities.

---

## The scaling law loop

The four current laws (#2 pre-training, #3 post-training, #4 inference, #5 agentic) are not independent — they feed each other in a cycle:

```
Pre-training → Post-training → Inference (harness leverage) → Agentic (real-world deployment)
       ↑                                                                    │
       └─────── Curation: "this is good enough to memorize" ───────────────┘
```

Each layer's output is candidate training data for the next. The closing step is **curation**: deciding which traces, decisions, and outcomes are worth feeding back into pre-training to start the loop from a higher platform. This is where the bottleneck increasingly lives — the labs that can identify novel, high-signal, non-redundant interaction data and route it back into training compound faster than those that can't.

The two substrates this loop depends on:

- **Simulation fidelity** — rate-limits how much interaction data can be generated, especially for embodied / robotic loops where real-world experiment rate is the constraint
- **Curation infrastructure** — rate-limits how much of the generated data is actually useful

These are in a chicken-and-egg relationship: sim fidelity must improve before curation can be validated, but curation must be ready before the data flood arrives. The window to build curation infrastructure is the gap between "sim is good enough to flood" and "the flood arrives."

---

## Cross-references in the dream state

| Topic | Dream state location |
|---|---|
| Compute deployment trajectory (External API → Sovereign local → Sovereign cloud → Space-based) | Deferred log: Compute infrastructure trajectory |
| Open hardware stack (RISC-V, Yosys, OpenLane, Skywater 130nm, Tiny Tapeout) | Deferred log: Open hardware stack — substrate sovereignty trajectory |
| Sovereign model training pipeline + data flywheel + trace triage | Deferred log: Sovereign model registry |
| Verifier's law | Architecture: Domain verifiability spectrum |
| CIEL framework (Capital / Intelligence / Energy / Labor) | Orient frameworks |
| Tech-tree framework (capability thresholds, recognition, runway preparation) | Orient frameworks |
| Bottleneck-targeted benchmarks via game design | Design Principles |
| Compound loop (system builds itself) | Loop Library section |
| The harness is the differentiator | Design Principles |

---

## Verification status

The structural claims (eras, MECE categories, named laws, paper authors) are accurate to standard public knowledge. The specific *numerical* claims marked `[verify]` come from the source conversation and should be checked against the original papers before being treated as fact:

- Kaplan ratios for compute-optimal allocation
- Llama token-to-parameter ratios in the "beyond-Chinchilla" range
- DeepSeek-R1 token multiplier vs. baseline reasoning models

The "future scaling laws" section is explicitly speculative. Categories 6–11 are candidates, not consensus, and the overlaps noted are real.
