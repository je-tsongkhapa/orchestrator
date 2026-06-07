# MECE Components of a Machine Learning System

**Partitioning axis: architectural planes** (not lifecycle — lifecycle has feedback loops that break mutual exclusion). Each component has exactly one home.

## 1. Data Plane — *manages records*
- **Sources & ingestion** — connectors, CDC, event streams, batch pulls
- **Storage** — raw (lake), processed (warehouse), online (KV/cache)
- **Labeling** — annotation UI, weak-supervision, programmatic labeling
- **Validation** — schema checks, distribution checks, quality SLAs
- **Lineage & versioning** — dataset snapshots, hashes, provenance

## 2. Feature Plane — *manages signals*
- **Transformation** — batch pipelines, streaming pipelines, on-demand
- **Feature store** — offline store, online store, materialization
- **Consistency layer** — training/serving skew prevention, point-in-time joins
- **Feature catalog** — discovery, ownership, semantics

## 3. Model Plane — *manages artifacts*
- **Training orchestrator** — distributed/parallel strategies, checkpointing
- **Experiment tracker** — runs, metrics, hyperparameters, code hashes
- **Hyperparameter search** — Bayesian/grid/PBT
- **Evaluator** — offline metrics, slice analysis, fairness/robustness
- **Model registry** — versions, stages (dev/staging/prod), signatures

## 4. Serving Plane — *manages predictions*
- **Inference runtime** — online (low-latency), batch, streaming, edge
- **Deployment controller** — canary, shadow, A/B, blue-green, rollback
- **Request gateway** — routing, auth, rate limit, input validation
- **Post-processing** — calibration, thresholding, business logic overlay
- **Cache & memoization** — embedding cache, response cache

## 5. Observability Plane — *manages truth about the system*
- **System metrics** — latency, throughput, error rate, cost
- **Data/feature monitors** — drift, skew, null-rate, cardinality
- **Model monitors** — accuracy decay, calibration drift, prediction distribution
- **Logs & traces** — request traces, prediction logs, lineage links
- **Alerting** — anomaly detection, SLO burn alarms

## 6. Feedback Plane — *manages the learning loop*
- **Ground-truth capture** — delayed labels, outcomes, conversions
- **Human-in-the-loop** — review queues, escalation, active learning selectors
- **Retraining trigger** — schedule-based, drift-based, performance-based
- **Label propagation** — joining feedback to prior predictions

## 7. Control & Governance Plane — *manages policy*
- **Workflow orchestrator** — DAGs, schedulers, retries, backfills
- **Access control** — RBAC, dataset/model permissions
- **Audit & compliance** — PII handling, consent, right-to-forget, model cards
- **Cost/quota governance** — budgets, accelerator quotas

## 8. Infrastructure Plane — *manages substrate*
- **Compute** — CPU/GPU/TPU pools, autoscaling, spot/on-demand mix
- **Storage substrate** — object, block, vector DB
- **Network** — ingress, service mesh, inter-AZ/region transport
- **Secrets & config** — key mgmt, config store

---

## Completeness check (what's explicitly placed, not missing)

- *Vector DB* → Infra (storage substrate); its semantic indexing role → Feature Plane (online store for embeddings)
- *RLHF / preference data* → Feedback Plane (human-in-the-loop) + Data Plane (labeled dataset)
- *Prompt/template registry* (for LLM systems) → Model Plane (registry sub-type, artifact = prompt)
- *Guardrails / safety filters* → Serving Plane (post-processing) + Observability (monitors)
- *MLOps tooling* is not a plane — it's the *cross-plane automation surface* spanning Control + Observability + Model

## What's intentionally excluded from "ML system"

Business application UI, downstream analytics, org/team structure. Those consume the system but aren't components of it.

---

# MECE Types of Machine Learning Systems

**Partitioning axis: the primary question the system answers / output structure.** (Not algorithm — a "recommender" can be collaborative filtering, deep learning, or RL; the *type* is defined by what it produces, not how.)

## 1. Descriptive systems — *"what structure is in this data?"*
Clustering, topic modeling, community detection, dimensionality reduction, segmentation, unsupervised anomaly discovery.

## 2. Predictive systems — *"what label or value does this input have?"*
Fraud detection, churn/propensity, credit scoring, medical diagnosis, content moderation, spam filtering, supervised anomaly classifiers.

## 3. Forecasting systems — *"what will this value be at a future time?"*
Demand forecasting, capacity planning, financial/price forecasting, weather, energy load, epidemiological projection.

## 4. Perception systems — *"what entities/properties are in this raw signal?"*
Object detection, semantic/instance segmentation, OCR, ASR, NER/parsing, pose estimation, sensor fusion, biometric recognition.

## 5. Retrieval / Ranking / Recommendation — *"which items from a corpus best match this query or context?"*
Web/product search, recommender systems (feed, next-video, product), ad targeting, RAG retrievers, semantic search.

## 6. Matching systems — *"which pairs should be joined?"*
Marketplace matching (riders↔drivers), entity resolution/dedup, record linkage, dating apps, knowledge-graph link prediction.

## 7. Generative systems — *"produce a new artifact that satisfies this spec."*
Text/image/audio/video generation, code generation, synthetic data, molecule/protein design, translation, summarization.

## 8. Decision / Control systems — *"what action should be taken now?"*
RL controllers, autonomous driving planners, robotics policies, dynamic pricing/bidding, algorithmic trading execution, thermostat/HVAC control.

## 9. Agentic / Planning systems — *"what sequence of actions achieves this goal?"*
Task-executing LLM agents, coding/research agents, workflow-automation agents, multi-agent orchestrators, AI tutors with curricula.

## 10. Simulation / World-model systems — *"what would happen under these conditions?"*
Physics/climate/epidemic simulators, digital twins, neural surrogates for PDEs, learned world models for model-based RL, game-engine NPC simulators.

## 11. Optimization / Design systems — *"what configuration maximizes the objective under constraints?"*
Bayesian optimization services, AutoML, neural architecture search, experimental design, hyperparameter tuning platforms, supply-chain/routing optimizers with ML surrogates.

---

## Overlap-resolution rule (for systems that seem to span two types)

When a real system spans types, classify by its **output to the consumer**, not its internal mechanism:

- ChatGPT = **Generative** (output = text). Its agentic tool-use is *mechanism*, not *type* — when Claude Code executes multi-step tool sequences to complete a user goal, that's **Agentic**.
- A deep-RL recommender = **Ranking** (output = ordered item list). RL is *how*, not *what*.
- A vision model that outputs bounding boxes = **Perception**, not Predictive, because the output carries spatial/structural content beyond a label.
- A forecasting model used inside a trading bot = two systems: Forecasting (produces a price estimate) feeding Decision (chooses an order). Compose, don't collapse.

## Composition note

Real products are almost always **compositions** of these types. A modern e-commerce stack = Perception (product image tagging) + Predictive (fraud) + Retrieval (search) + Ranking (recs) + Forecasting (inventory) + Decision (dynamic pricing). The MECE list applies per subsystem, not per product.

## What's intentionally excluded

- *ML-adjacent systems* (analytics dashboards, A/B testing platforms, feature flag services) — they consume or enable ML but don't answer an ML question themselves.
- *Modality labels* (vision, NLP, speech, tabular) — orthogonal axis; every type above can appear in every modality.
