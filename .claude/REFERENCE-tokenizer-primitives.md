# Tokenizer Primitives — BOS, EOS, and Reserved Tokens

> Canonical reference for tokenizer-layer structural primitives and their use as capability levers in agent systems. Six use families from normal inference control to adversarial exploit surfaces. Generalizes from BOS/EOS to custom reserved tokens for fine-tuning (phase tokens, boundary tokens, provenance tokens).
>
> Revisit when: evaluating sovereign model registry / vLLM deployment (dream-state deferred item) — this doc is the pre-built reference for custom-token design at fine-tuning time; reviewing seed-data sanitization at any app-layer boundary that flows user input into LLM prompts; a special-token exploit surface becomes practically relevant; designing boundary markers for agentic runtime extensions (phase tokens, tool-call tokens, provenance tokens).

---

## What BOS and EOS actually are

At the tokenizer level, a language model processes a stream of integer token IDs. Two of those IDs are conventionally reserved as structural markers:

- **BOS (beginning of sequence)** — `<s>`, `<|begin_of_text|>`, `<|startoftext|>`, etc. Signals "a new sequence starts here."
- **EOS (end of sequence)** — `</s>`, `<|end_of_text|>`, `<|endoftext|>`, etc. Signals "this sequence ends here."

These aren't text — they're atomic tokens the model learned specific behaviors around during training. They're part of the same vocabulary as regular word tokens, just reserved.

**Where they come from.** The tokenizer's training data was structured with these tokens at document boundaries. Pre-training saw billions of sequences each wrapped in `<BOS>...<EOS>`. The model learned:

- After `<BOS>`: "I'm starting fresh, draw on general priors, no prior conversation."
- Before `<EOS>`: "This document/turn has a complete shape; things of certain lengths and structures end here."
- Generating `<EOS>`: "I'm done; stop."

Modern chat models extend this with role markers — `<|im_start|>`, `<|im_end|>`, `<|user|>`, `<|assistant|>`, `<|system|>` — which serve the same structural purpose at a finer granularity. In Llama-3's tokenizer, the separation is:

- `<|begin_of_text|>` — BOS for the whole sequence
- `<|start_header_id|>role<|end_header_id|>` — role delimiters
- `<|eot_id|>` — end-of-turn (not end-of-text)
- `<|end_of_text|>` — true EOS

---

## Six use families

Roughly ordered from normal to adversarial.

### 1. Inference control (normal use)

When you call a chat model via API, BOS/EOS are handled by the chat template — you don't touch them. When you call a completion model (or a raw model via llama.cpp, vLLM, etc.), you choose whether to prepend BOS and whether to stop generation on EOS:

- **Forcing EOS as a stop condition** — standard; set `stop_sequences` or `eos_token_id` to truncate generation cleanly.
- **Forcing a custom EOS** — in fine-tuning, you can pick a custom stop token so the model's natural output ends at your chosen boundary. Widely used in code models (stop on `<|endoffile|>`), agentic models (stop on `<|tool_call_end|>`).
- **Suppressing EOS for longer generations** — rare but real; sometimes the model wants to stop too early, and forcibly lowering the EOS token's logit extends generation.

### 2. Prompt structuring (normal-to-clever)

If you have direct tokenizer access (open-weights / local models), structuring prompts with explicit BOS/EOS boundaries to mimic the pretraining shape can improve quality:

- **"Document completion" prompting** — wrap each example in `<BOS>example<EOS>`, then let the model complete the next one. Exploits the model's learned "document" prior.
- **Few-shot separation** — using `<EOS>` between shots is often cleaner than visible delimiters like `###`, because the model treats `<EOS>` as a hard break rather than something to stylistically continue.
- **System-vs-content separation** — in raw completion with a chat model, placing the system prompt before BOS of the "user message" can signal structural priority.

### 3. Fine-tuning tricks (normal, load-bearing)

Most practical family:

- **Custom tokens for agent boundaries.** When fine-tuning an agentic model, introducing `<|tool_call|>` / `<|tool_result|>` / `<|plan|>` as reserved tokens gives you stop points and role markers cheaper than text-based parsing. The model learns to emit them at the right moments; your runtime can sample-until-`<|tool_call|>`, execute, and feed results back.
- **Structured-output fine-tuning.** Reserve `<|schema_start|>` / `<|schema_end|>` for JSON generation so the model has a clean "I'm inside structured output now" marker. Cheaper and more robust than begin/end sentinel strings.
- **Chain-of-thought boundary tokens.** OpenAI's o-series and similar reasoning models use hidden reasoning tokens that emit between `<|thinking|>` / `<|end_thinking|>` and are stripped before showing the user. Achievable in open-weights models through fine-tuning.

### 4. Stop-token hacks (clever)

- **Multi-stop agentic loops.** Configuring the sampler to stop on `<|tool_call_end|>` OR `<|eot_id|>` OR a custom `<|yield_to_user|>` lets a single generation loop handle tool calls, end-of-turn, and conversational yields without state management.
- **Streaming structured output.** Streaming until a reserved token instead of a string match gives O(1) deterministic detection of structure boundaries mid-stream, instead of buffering for a maybe-partial string match.

### 5. Attention / context exploits (research territory)

- **Attention sink theory.** Research has shown that the first few tokens of a sequence (often BOS-adjacent) accumulate disproportionate attention weight regardless of content, becoming "attention sinks" that stabilize generation. Two consequences:
  - **In streaming LLMs** (Xiao et al., "Efficient Streaming Language Models with Attention Sinks," 2023), retaining the BOS token when sliding the context window prevents degradation — you keep the sink, discard the rest.
  - **For adversarial purposes**, knowing BOS is an attention sink gives a theoretical target for prompt injection via special tokens.
- **Logit-lens work.** Models often encode "I am starting fresh" state in hidden representations immediately after BOS. Probing / steering vectors extracted from these positions can be more reliable than mid-sequence probes.

### 6. Adversarial / exploit territory (careful)

- **Special-token injection in user input.** If an API doesn't sanitize user input before rendering it into the chat template, a user who pastes the literal string `<|endoftext|>` or `<|im_end|>` can break out of their own turn. The model sees what looks like a turn boundary and can be tricked into generating a fake system or assistant turn. Called "chat template injection" or "special token injection." Most hosted APIs sanitize this server-side (Anthropic, OpenAI strip/escape special-token strings from user content before template rendering). Local / self-hosted deployments *often don't* — a real bug surface.
- **Premature EOS to truncate responses.** Injecting a pattern that makes the model more likely to emit EOS early — e.g., prompts ending with "In summary:" can reliably end responses sooner. Not strictly special-token specific; the model learned that "In summary:" precedes wrap-up patterns.
- **BOS attention-sink hijacking.** Research-stage: can a prompt be crafted such that the BOS-adjacent attention-sink position carries adversarial steering? Some success in academic papers (Geiping et al. on embedding-space attacks — not directly BOS but related). Not a practical exploit vector against frontier APIs today.
- **EOS suppression in refusal bypasses.** If a refusal model is trained to generate `I can't help with that<EOS>`, some jailbreaks work by forcing generation past the EOS with logit bias or by structuring prompts that make EOS less likely after the refusal, causing the model to keep generating past its trained refusal boundary.
- **Token ID collisions in tokenizers.** Some tokenizers have undertrained tokens (the "SolidGoldMagikarp" phenomenon, 2023) — IDs that appear rarely in training data and have strange, unpredictable model behavior. Not BOS/EOS specifically, but same family: structural-level tokenizer weirdness that can be exploited.

---

## Practical implications

### Near-term, frontier-API-only work

Nothing directly exploitable. Using Anthropic / OpenAI via their APIs, BOS/EOS is handled server-side and special tokens are sanitized from user input before template rendering. Trying to exploit them at the API level is a dead end — local open-weights is the prerequisite.

### When sovereign model registry / vLLM lands

BOS/EOS and custom reserved tokens become substantial capability levers. Specific applications for an orchestrator-style harness:

- **Custom agent-role tokens for shadow clones.** Instead of text-based turn markers (`---CLONE-BOUNDARY---`), reserve a `<|clone_yield|>` token during fine-tuning. Cheaper to parse, deterministic to detect mid-stream, impossible for user input to spoof.
- **Loop-phase tokens.** `<|observe|>`, `<|orient|>`, `<|decide|>`, `<|act|>` as reserved tokens during fine-tuning means the model can emit phase transitions directly. Runtime hooks into these for phase tracking without post-hoc parsing.
- **Attention-sink retention in long-context sessions.** For long autonomous loops using context-window slicing, retaining BOS is free insurance against degradation per the streaming-LLM paper. Frontier APIs handle this internally; local deployments need explicit handling.
- **Structured-output fine-tuning for artifact emission.** Reserve `<|artifact_start|>` / `<|artifact_end|>` for the one-artifact-per-loop commitment; artifact extraction becomes sub-millisecond deterministic.
- **Provenance tokens.** `<|from_memory|>`, `<|from_retrieval|>`, `<|from_user|>` that mark sources inline and propagate through attention. Provenance becomes queryable at the token level.

### Security implications for app-layer LLM integrations

When an app accepts user-provided text fields (descriptions, reviews, comments, etc.) and those flow into LLM prompts, a malicious user could inject special-token strings hoping to escape the template. Defenses:

- **Strip or escape special-token patterns from user input** before prompt construction (`<|im_end|>`, `</s>`, `<|endoftext|>`, Claude's `\n\nHuman:` / `\n\nAssistant:` equivalents, etc.)
- **Use structured API calls** (system/user/assistant role objects) rather than raw string concatenation, so the template renderer handles escaping.

Frontier APIs sanitize server-side by default, but belt-and-suspenders sanitization at the app-layer edge function is still cheap and defensive. Worth a small hardening pass anywhere user-controlled strings flow into prompts.

---

## The generative framing

**BOS/EOS are the simplest case of a general pattern: reserved tokens as structural primitives the model can learn semantics around.** The generalization is rich:

- **Phase tokens** for OODA cycles or any multi-phase protocol
- **Boundary tokens** for artifact / tool / clone emission
- **Meta tokens** for model behavior switches (verbose / terse, thinking / direct, creative / literal)
- **Provenance tokens** (`<|from_memory|>`, `<|from_retrieval|>`, `<|from_user|>`) that mark sources inline and propagate through attention

**The cost**: fine-tuning access is required to teach the model what the reserved tokens *mean*. Without that, injecting novel special tokens just produces gibberish — the model has no prior for what `<|clone_yield|>` does, so it treats the raw bytes as noise. Fine-tuning is where these tokens earn their structural privilege; you're teaching the model "here is a special category of token that modifies generation behavior."

**The payoff**: once taught, reserved tokens give you deterministic, sub-millisecond, impossible-to-spoof structural signals — a cleaner substrate than string-based delimiters at every level of an agent runtime.

---

## Connections

- **Sovereign model registry / vLLM** (dream-state deferred item in `DREAM-STATE-ooda.md`) — this doc is the pre-built reference for custom-token design at the moment that lands.
- **Attention sinks** (Xiao et al. 2023, *Efficient Streaming Language Models with Attention Sinks*) — the streaming-LLM paper. Load-bearing for long-context retention in local deployments.
- **Undertrained tokens / tokenizer weirdness** — the SolidGoldMagikarp family (2023). Same structural category (special behavior tied to specific token IDs).
- **Embedding-space adversarial attacks** — Geiping et al. and related literature. Attacks on the attention-sink position rather than on BOS directly, but adjacent.
- **Prompt-injection discipline** — the chat-template-injection family of exploits. Defensive practice at every app-layer boundary that flows user input into LLM prompts.
- **Chain-of-thought reasoning tokens** — how o-series and similar reasoning models use hidden `<|thinking|>` tokens; reference architecture for any local CoT implementation.
- **Loop artifact discipline** — one-artifact-per-loop commitment in the harness. `<|artifact_start|>` / `<|artifact_end|>` is the tokenizer-level implementation if/when the orchestrator goes local.

---

## What this doc is for

Pre-compiled reference for the moment tokenizer-layer primitives become practically relevant — specifically, when the sovereign model registry lands and fine-tuning of in-house agentic models begins. The content is stable (tokenizer mechanics don't change fast); the payoff is cheap second invocation ("what were the custom-token tricks we flagged?") without re-deriving from scratch.
