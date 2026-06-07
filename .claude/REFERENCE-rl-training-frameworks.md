# RL Training Frameworks Comparison

> Source: HuggingFace blog by Amine Dirhousssi (2026). Compares 16 RL training frameworks.
> Revisit when: sovereign model registry is active and RL training is being implemented.

## Comparison Dimensions

| Dimension | What it evaluates |
|-----------|-------------------|
| **Orchestration & Concurrency Primitive** | How distributed components are coordinated (Ray actors, asyncio, pub/sub, HTTP) |
| **Rollout Buffer Design** | How rollouts flow from inference to training |
| **Weight Synchronization Protocol** | How updated weights reach inference servers, and whether the system must pause to accept them or continue generating |
| **Staleness Management** | How off-policy rollouts are handled: version rejection, depth bounding, or importance-sampling correction |
| **Partial Rollout Handling** | What happens to in-flight generations when a weight update arrives mid-sequence |
| **LoRA Training Support** | Whether adapter-only parameters can be trained and synced, enabling sub-millisecond weight transfers |
| **Distributed Training Backend & Parallelism** | What parallelism strategy is used for training, constraining max model size |

## Frameworks (sorted by GitHub stars, Mar 2026)

| Library | Organisation | Repo | Stars |
|---------|-------------|------|-------|
| **VeRL** | ByteDance | github.com/verl-project/verl | 19,673 |
| **ART** | CoreWeave | github.com/OpenPipe/ART | 8,952 |
| **SLIME** | THUDM | github.com/THUDM/slime | 4,595 |
| **AReaL** | inclusionAI/Ant Group | github.com/inclusionAI/AReaL | 4,338 |
| **verifiers-rl** | PrimeIntellect | github.com/PrimeIntellect-ai/verifiers | 3,876 |
| **open-instruct** | AI2 (AllenAI) | github.com/allenai/open-instruct | 3,611 |
| **ROLL** | Alibaba | github.com/alibaba/ROLL | 2,921 |
| **Tunix** | Google | github.com/google/tunix | 2,175 |
| **SkyRL** | NovaSky-AI | github.com/NovaSky-AI/SkyRL | 1,664 |
| **NeMo-RL** | NVIDIA | github.com/NVIDIA-NeMo/RL | 1,383 |
| **PRIME-RL** | PrimeIntellect | github.com/PrimeIntellect-ai/prime-rl | 1,114 |
| **MILES** | radixark | github.com/radixark/miles | 950 |
| **Atropos** | NousResearch | github.com/NousResearch/atropos | 878 |
| **OAT** | SAIL-SG | github.com/sail-sg/oat | 637 |
| **TorchForge** | Meta | github.com/meta-pytorch/torchforge | 632 |
| **PipelineRL** | ServiceNow | github.com/ServiceNow/PipelineRL | 374 |

## Selection Notes

When choosing a framework, key decision factors for the harness:

- **VeRL** (ByteDance) — most popular, likely best community support and documentation
- **ART** (CoreWeave/OpenPipe) — strong infrastructure backing
- **open-instruct** (AI2) — research-oriented, good for experimentation
- **verifiers-rl** (PrimeIntellect) — specifically relevant to our Lean proof verification interest (verifier-based RL)
- **NeMo-RL** (NVIDIA) — enterprise-grade, likely best GPU optimization

The sovereign model registry entry specifies: "RL-based training is more parameter-efficient than SFT for this (even tiny parameter counts improve RL performance), favoring RL approaches for sovereign model training." Framework selection should prioritize LoRA support (sub-millisecond weight transfers for efficient fine-tuning) and staleness management (critical for concurrent agent workloads).
