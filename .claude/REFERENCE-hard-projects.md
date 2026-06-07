# MECE List of "Hard" Programming Projects

**Definition of "hard":** the project forces you to confront a fundamental CS/systems difficulty that can't be handwaved — not merely ambitious in scope.

**Partitioning axis: the *primary bottleneck skill* the project forces you to master.** Projects that touch multiple categories are classified by where difficulty actually bites.

## 1. Operating systems & low-level systems
- Microkernel or Linux-clone kernel
- Memory allocator (malloc replacement, moving GC)
- Journaling or copy-on-write filesystem
- Process scheduler or container runtime
- Driver for real hardware

*Hard because: hardware interface, no safety net, debugging without the OS you're writing.*

## 2. Compilers & language design
- Self-hosting compiler for a new language
- Bytecode VM with optimizing JIT
- Static type checker (Hindley-Milner, gradual typing, dependent types)
- Full-feature regex engine (Unicode, backrefs, zero-width)
- Package manager with SAT-based dependency resolution

*Hard because: correctness over an infinite input space.*

## 3. Databases & storage engines
- B-tree or LSM-tree storage engine
- Query planner/optimizer with cost model
- Transaction manager (MVCC, serializable isolation)
- Columnar analytics engine with vectorized execution
- WAL + crash recovery surviving pulled power

*Hard because: ACID under arbitrary failure.*

## 4. Distributed systems
- Raft or Multi-Paxos implementation with membership changes
- Distributed KV store (consistent hashing, leader election, replication)
- CRDT store with causal consistency
- Distributed lock manager with fencing tokens
- Gossip + phi-accrual failure detector

*Hard because: partial failure, asynchronous network, FLP impossibility.*

## 5. Concurrency & parallel runtimes
- Async runtime with work-stealing scheduler
- Lock-free queue, hashmap, or memory reclamation (hazard pointers)
- Actor system with supervision
- Software transactional memory
- Green-thread scheduler with preemption

*Hard because: memory model reasoning; sequential intuition fails.*

## 6. Networking protocols
- TCP from scratch in userspace (retransmission, congestion control)
- HTTP/3 or QUIC implementation
- WebRTC stack (ICE, DTLS-SRTP)
- BitTorrent client with DHT (Kademlia)
- DNS resolver with DNSSEC validation

*Hard because: adversarial network + spec fidelity against real peers.*

## 7. Cryptography & security
- TLS 1.3 from primitives (no OpenSSL)
- Signal-protocol encrypted messenger
- Zero-knowledge proof system (Groth16, STARK)
- Reproduction of a known CVE in a sandbox
- Password manager with sound KDF + sync

*Hard because: one bit wrong = broken, and you won't notice.*

## 8. Graphics & rendering
- Physically-based path tracer with BVH
- Real-time renderer on modern Vulkan/Metal
- Signed-distance-field raymarcher
- Monte Carlo denoiser
- Volumetric renderer (clouds, fog)

*Hard because: math + GPU parallelism + sub-16ms budget.*

## 9. Physical & numerical simulation
- Rigid-body physics engine with stable contacts
- SPH or Eulerian fluid sim
- N-body gravitational simulator (Barnes-Hut)
- Finite-element solver
- Cloth/hair/soft-body simulator

*Hard because: numerical stability + conservation laws + scale.*

## 10. Scientific / numerical primitives
- BLAS/LAPACK subset, cache-tuned
- FFT library (mixed-radix, Bluestein)
- Autodiff engine (forward + reverse mode)
- SAT or SMT solver
- LP/MIP solver (simplex + branch-and-cut)

*Hard because: numerical accuracy + cache-aware performance.*

## 11. Machine learning systems
- Deep-learning framework with graph autograd
- Transformer training from scratch on commodity GPUs
- LLM inference engine with paged KV-cache and speculative decoding
- Distributed training orchestrator (tensor/pipeline/ZeRO parallel)
- Vector database with ANN indexing (HNSW, IVF-PQ)

*Hard because: research-engineering frontier; correctness signals are statistical.*

## 12. Information retrieval & search
- Inverted-index search engine with BM25 + learned re-ranking
- Hybrid dense+sparse retrieval with fusion
- Web-scale crawler with politeness + dedup
- Query autocomplete with spell correction
- Learning-to-rank pipeline end-to-end

*Hard because: relevance is fuzzy; scale amplifies every bug.*

## 13. Hardware-adjacent & embedded
- CPU emulator (Game Boy, NES, RISC-V, x86 subset)
- FPGA project in Verilog/VHDL (e.g., GPU, CPU, MIPI)
- Real-time control loop on microcontroller (motor, drone)
- USB or PCIe device implementation
- Logic placer / synthesizer tool

*Hard because: timing, physical constraints, non-determinism.*

## 14. Program analysis & verification
- Static analyzer via abstract interpretation
- Symbolic execution engine
- Model checker over temporal logic (TLA+, SPIN-like)
- Coverage-guided fuzzer with instrumentation
- Formally verified data structure in Coq / Lean / F*

*Hard because: reasoning over all executions, not just one.*

## 15. Real-time interactive systems
- Multiplayer netcode with rollback + lag compensation
- Deterministic ECS game engine
- Low-latency audio engine (lock-free DSP graph)
- Collaborative editor with OT or CRDT sync
- Text editor from scratch — rope or piece table, incremental re-parse (tree-sitter-style), modal editing, undo stack as algebra, LSP integration under sub-frame latency
- VR/AR compositor with predictive reprojection

*Hard because: strict latency budget + interaction correctness + determinism.*

## 16. Reactive / incremental computation systems
- Spreadsheet — formula language, dependency graph, topological recompute, cycle detection, volatile functions, array formulas, sparse evaluation
- Build system with hashed-input incremental builds (Make/Bazel-class) — content addressing, remote cache, correct minimality
- Incremental computation framework (Adapton / Salsa / rust-analyzer's on-demand engine) — demand-driven memoization, fine-grained invalidation
- Reactive UI runtime — virtual-DOM diff + fiber scheduler, concurrent rendering, suspense
- Live notebook / reactive programming environment (Observable-class) — topological cell re-evaluation, generator semantics

*Hard because: correctness of the dependency graph under mutation + minimality of recomputation + glitch-free propagation. Wrong = stale results nobody sees until it matters.*

---

## Overlap-resolution rule

Classify by the **skill bottleneck**, not surface area.

- "Write a database on top of your own filesystem" = **Databases** (transactions are what breaks you), not OS.
- "LLM agent that controls a browser" = **ML systems** (inference/tooling) unless the novelty is the multi-step planning, in which case it's an agentic **real-time interactive** system.
- *Spreadsheet* vs *compiler*: the formula-evaluator inside a spreadsheet is a mini-language (compiler territory), but the project's bottleneck is the reactive graph, not the parser. Classify as **Reactive**.
- *Build system* vs *distributed*: a distributed build farm touches category 4, but the core difficulty is still incremental correctness. Classify as **Reactive**.

## Intentionally excluded

- **Scope-hard, not CS-hard**: another bigger CRUD app, a larger microservice mesh.
- **Domain-complexity-hard**: tax software, accounting systems — hardness is requirements capture, not computation.
- **Glue projects**: integrations, scrapers, dashboards — hardness is API archaeology, not CS.

## Picking one

Pick the category where you're **weakest** and where the bottleneck scares you a little. Completing a hard project in a category you already know is learning consolidation, not stretch.
