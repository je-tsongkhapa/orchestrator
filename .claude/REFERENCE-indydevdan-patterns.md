# IndyDevDan Implementation Patterns

> Source: IndyDevDan (YouTube channel + GitHub repos). Implementation-level patterns for agentic engineering on Claude Code.
> This is a living reference — add new patterns as resources are processed.
> Revisit when: implementing Act's coordination, building agent teams, or designing validation infrastructure.

## Why This Reference Exists

IndyDevDan independently builds patterns that map to our architecture at the implementation level. His work is consistently concrete, code-level, and immediately reusable — making it the strongest external reference for how to implement what we've designed.

## Abstraction Mapping

His primitives are one level down from ours — the raw materials vs. the reusable structures built from them.

| His primitive | Definition | Our abstraction |
|---------------|-----------|-----------------|
| Context | What the agent knows | Memory surfaces |
| Model | Which LLM runs | Model selection dimension in Decide |
| Prompt | The instruction | Skill definitions |
| Tools | What the agent can do | Tool scoping in isolation layers |

| His leverage point | Definition | Our abstraction |
|--------------------|-----------|-----------------|
| Reusable prompts | Commands run repeatedly | Skills as the atomic primitive |
| Specialized agents | Focused context, one job | Loop-bound sub-agents |
| AI Developer Workflows | Multi-step pipelines | Loops with phases and gates |
| Hooks | Deterministic validation | Per-skill co-located hooks |
| Template metaprompts | Prompts that generate prompts | Meta-loop |

---

## Reliability Model: Five Reinforcing Layers

The meta-pattern across all of his work: **never trust a single mechanism.** Five layers, each catching what the others miss:

| Layer | Mechanism | What it does |
|-------|-----------|-------------|
| **1. Structural constraints** | Frontmatter `disallowedTools`, tool matchers on hooks, `disallowed-tools` on prompts | Prevent bad actions — validators can't write, planners can't execute, validation only fires on relevant operations |
| **2. Deterministic validation** | Co-located hook scripts (Python, not LLM), JSON I/O convention, two granularities (PostToolUse + Stop) | Catch bad output — `ruff check`, `ty check`, CSV parsing, file-exists checks, required-section checks |
| **3. Templating** | AGP code snippet, template metaprompts, `validate-file-contains` enforcing required sections | Ensure consistent format — output structure is fixed and validated, not just suggested |
| **4. Separation of concerns** | Builder/validator split, plan/execute split, what/how split, one task per agent | Focus agents — each agent has a narrow context window doing one thing well |
| **5. Observability** | Per-validator log files, task system status, saved plan artifacts in `specs/`, TTS on completion | Prove it worked — inspectable evidence at every layer |

Any one layer failing is caught by another. Structural constraints prevent bad actions the validator would catch. Templates standardize what the validator checks. Separation ensures the validator isn't the same agent that built the artifact. Logging proves all four layers ran.

---

## 1. Validation

**Dream state mapping:** per-skill deterministic validation (line ~1176), trust strategies (Eliminate)

### Hook I/O Convention

All validators are UV single-file Python scripts. Read JSON from stdin (hook input), output JSON to stdout (`{"result": "continue"}` or `{"result": "block", "reason": "..."}`), log to `<script>.log` in same directory. PostToolUse validators extract `tool_input.file_path` from hook input and skip non-relevant files (return `{}`).

### Co-Located Hooks in Frontmatter

Each prompt/agent/skill carries its own validation in YAML frontmatter. Validation travels with the capability.

**Builder agent (PostToolUse — micro-validation):**
```yaml
hooks:
  PostToolUse:
    - matcher: "Write|Edit"
      hooks:
        - type: command
          command: uv run $CLAUDE_PROJECT_DIR/.claude/hooks/validators/ruff_validator.py
        - type: command
          command: uv run $CLAUDE_PROJECT_DIR/.claude/hooks/validators/ty_validator.py
```

**Metaprompt (Stop — structural validation):**
```yaml
hooks:
  Stop:
    - hooks:
        - type: command
          command: >-
            uv run $CLAUDE_PROJECT_DIR/.claude/hooks/validators/validate_new_file.py
            --directory specs --extension .md
        - type: command
          command: >-
            uv run $CLAUDE_PROJECT_DIR/.claude/hooks/validators/validate_file_contains.py
            --directory specs --extension .md
            --contains '## Task Description'
            --contains '## Team Orchestration'
            --contains '### Team Members'
```

### Builder/Validator as Minimum Viable Team

2x compute for every task as the default, not an optimization. Builder does the work with PostToolUse micro-validation (formatters, type checkers); validator checks the completed artifact at macro-level (does the output satisfy requirements). Validator has `disallowedTools: Write, Edit, NotebookEdit` — read-only by design.

### Quality Gate Hooks (Agent Teams Platform)

`TeammateIdle` (runs when teammate about to idle — exit code 2 sends feedback and keeps working), `TaskCreated` (exit code 2 prevents creation), `TaskCompleted` (exit code 2 prevents completion and sends feedback). The platform's equivalent of our gate approvals.

**Validator directory convention:** `.claude/hooks/validators/` with individual scripts per concern. Each validator logs to its own file for observability.

**Reference repo:** github.com/disler/claude-code-hooks-mastery

---

## 2. Orchestration

**Dream state mapping:** Act module (shadow clones, static plan principle, execution engine)

### Task System

`TaskCreate`, `TaskGet`, `TaskList`, `TaskUpdate` — agents communicate through the shared task list. Tasks have dependencies, blocking, and completion events. Sub-agents ping back to the primary agent on completion — no polling, no bash sleep loops.

```
TaskCreate(subject, description) → taskId
TaskUpdate(taskId, addBlockedBy: [depIds]) → dependency ordering
TaskUpdate(taskId, status: "in_progress", owner: "agent-name") → assignment
Task(prompt, resume: agentId) → resume agent with preserved context
TaskUpdate(taskId, status: "completed") → triggers unblocking of dependents
```

### Agent Teams (Official Platform)

**Source:** code.claude.com/docs/en/agent-teams. Experimental: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`.

**Four components:** team lead (main session), teammates (separate Claude Code instances), task list (shared work items with dependencies), mailbox (SendMessage for inter-agent communication).

**Key details:**
- Teammates get independent context windows. They load project context (CLAUDE.md, MCP, skills) + spawn prompt, but NOT the lead's conversation history.
- Task dependencies resolve automatically: completing a task unblocks dependent tasks.
- Task claiming uses file locking to prevent race conditions.
- Any teammate can message any other by name — not just lead↔teammate.
- Display modes: `in-process` (same terminal, Shift+Down to cycle), `tmux` (split panes), `auto` (tmux if available).
- Full lifecycle: `TeamCreate` → `TaskCreate` (with dependencies) → spawn agents → parallel work → agents complete and `SendMessage` back → `TeamDelete` (cleanup + context reset).

**Limitations:** no nested teams, no session resumption with in-process teammates, one team per session, lead is fixed, permissions set at spawn time.

**Best practices:** 3-5 teammates, 5-6 tasks per teammate. Break work so each teammate owns different files.

### Template Metaprompt

A prompt that generates another prompt in a specific format. Three components: (1) self-validation hooks in frontmatter, (2) team orchestration section with agent assignments, (3) step-by-step task template with dependency ordering. Orchestration prompt (how to organize the team) is a separate input from the user prompt (what to build).

**Generated plan format (enforced by validators):**
```markdown
## Team Members
- Builder (SessionEnd Hook)
  - Name: session-end-builder
  - Role: Implement SessionEnd hook with logging
  - Agent Type: builder
  - Resume: true

## Step by Step Tasks
### 1. Implement SessionEnd Hook
- **Task ID**: session-end-hook-impl
- **Depends On**: none
- **Assigned To**: session-end-builder
- **Agent Type**: builder
- **Parallel**: true
```

### Higher-Order Prompts (HOPs)

A prompt that takes another prompt as a parameter — like a higher-order function. The outer prompt provides consistent workflow wrapping (save results, set up directories, report format); the inner prompt provides the specific steps. Maps to our loop/skill separation — the loop is the HOP, the skill is the inner prompt.

### Key Orchestration Patterns

- **Context refresh between planning and execution:** plan in one agent, kick off execution in a fresh agent with the plan as input.
- **Team deletion as context hygiene:** deleting the team after work forces a clean reset. Deliberate context engineering.
- **Ad hoc team spawning:** firing off teams mid-conversation to resolve issues that surface, not just from planned metaprompts.
- **31% primary context usage:** 8 parallel agents, orchestrator at 31% context. Sub-agent isolation preserves orchestrator capacity.
- **Dependency-ordered execution:** builders run in parallel, validators run after builders complete, final tasks after validation passes.

**Reference repos:** github.com/disler/claude-code-hooks-mastery

---

## 3. Observability

**Dream state mapping:** microscope principle (observability instruments), cockpit vocabulary

### Observability Dashboard

**Reference repo:** github.com/disler/claude-code-hooks-multi-agent-observability

**Data flow:** hook fires → `send_event.py` HTTP POSTs to `localhost:4000/events` → Bun server stores in SQLite + broadcasts via WebSocket → Vue 3 client renders real-time.

**Event capture (12 hook types):** every Claude Code hook type has two scripts chained in settings.json — the event-specific handler AND `send_event.py` which forwards to the dashboard server. `send_event.py` always exits 0 to never block Claude Code.

**Server (Bun + TypeScript, port 4000):**
- `POST /events` — receive from hooks, validate, store in SQLite, broadcast to all WebSocket clients
- `WS /stream` — on connect, sends 300 recent events; then streams new events in real-time
- SQLite with WAL mode for concurrent access

**Client (Vue 3 + Vite, port 5173):**
- `LivePulseChart` — canvas-based real-time timeline. Time windows: 1m/3m/5m/10m. Session-colored bars with event type + tool emoji overlays.
- `AgentSwimLane` — per-agent timeline strip keyed by `app:session`. Model badge, event count, tool count, avg gap, mini canvas chart.
- `EventTimeline` — scrollable event list with dual-color borders (app + session), auto-scroll, regex search.
- Max 300 events in memory (configurable via `VITE_MAX_EVENTS_TO_DISPLAY`), FIFO removal.

### Tmux Layout

Orchestrator pane on the left (wider), sub-agent panes stacked vertically on the right. Status bar: `@main @sbx-agent-1 @sbx-agent-2 ... · N teammates`. Each pane's status line: `[Opus 4.6] | # [########------] | 44.0% used | ~112.0k left | <session-id>`.

**Orchestrator narration:** the main agent provides running status commentary in its pane ("4 agents launched", "3 agents actively working, 1 about to start", "Both fixes complete. Let me grab the reports and shut down the agents."). The pilot reads the orchestrator pane to understand overall progress without inspecting individual agents.

### Status Line

Script (`status_line_v6.py`): reads model, session_id, context_window from stdin JSON. Renders: `[Opus 4.6] | # [########------] | 44.0% used | ~112.0k left | <session-id>`. Progress bar color-coded: green <50%, yellow 50-75%, red 75-90%, bright red >90%.

### TTS Integration

Notification hook priority chain: ElevenLabs Flash v2.5 (~75ms latency) > OpenAI TTS > pyttsx3 (offline fallback). Fires on permission prompts and non-generic idle prompts.

### Proof of Work

Agents take screenshots at each step as evidence. Deliverables include visual proof alongside the actual output. Each hook logs to its own file. Task system provides per-agent status.

---

## 4. Execution Environments

**Dream state mapping:** isolation layers, Linux as canonical execution environment, agent sandboxes (E2B/Daytona)

### Agent Sandbox Skill (E2B)

**Reference repo:** github.com/disler/agent-sandbox-skill

A Claude Code skill wrapping E2B's API into a CLI (`sbx`) for managing cloud sandboxes.

**Five command groups:**
- `sbx sandbox` — lifecycle: create, connect, info, pause, resume, extend-lifetime, kill, list, get-host
- `sbx files` — file ops: ls, read, write, edit, upload, download, upload-dir, download-dir
- `sbx exec` — command execution with cwd, env, root, shell, timeout, background
- `sbx browser` — Playwright automation: init, start, nav, eval, screenshot, click, type, a11y, dom, close
- `sbx init` — quick creation with template and timeout

**5 template tiers:** lite (2vCPU/4GB) → standard (4vCPU/4GB) → heavy (4vCPU/8GB) → max (8vCPU/8GB). Default: 2vCPU/2GB.

**Multi-agent safety:** capture sandbox ID in agent context (NOT shell variables — other agents overwrite). Default 12-hour timeout, 24-hour E2B Pro max.

**Orchestration pipeline:** Plan → Build → Host (expose port, get public URL) → Test (Playwright browser validation).

### Dedicated Agent Device (Mac Mini)

**Principle:** "If you want your agent to perform like you, give them the tools you have." A dedicated device where the agent has full OS control. The agent never shares the device with the pilot.

**Architecture (4 CLIs + 2 skills):**

| Component | Purpose | Implementation |
|-----------|---------|----------------|
| **Listen** | HTTP server waiting for job requests | Python, spawns Claude Code in tmux per job |
| **Direct** | Client CLI to call into listen server | Python, runs from pilot's machine |
| **Steer** | GUI control — click, type, screenshot, read screen | Swift CLI wrapping macOS accessibility APIs + OCR |
| **Drive** | Terminal control — spawn tmux panes, send/read commands | Python wrapper around tmux |

**Data flow:** trigger (just command / HTTP / Slack) → Direct → Listen server on agent device → spawns Claude Code → agent uses Steer (GUI) and Drive (terminal) → AirDrop results back to pilot.

**Key patterns:** YAML job system scalable to multiple devices, spec-driven tasks from `specs/` directory, time budgets (agent checks elapsed time and wraps up), self-cleanup after completion.

---

## 5. Composition

**Dream state mapping:** skills as atomic primitive, meta-loop, investment model

### Four-Layer Architecture

Each layer is independently useful — you can enter at any level to test, debug, or invoke.

| Layer | Purpose | What it adds |
|-------|---------|--------------|
| **1. Skills** | Raw capability | Wraps a CLI/tool into agent-readable form |
| **2. Agents** | Scale + specialization | Parallelization, context isolation, concrete workflow |
| **3. Commands** | Orchestration | Coordinates agents toward a common goal |
| **4. Justfile** | Reusability | One-line invocation with pre-configured flags |

### Agent Directory Convention

`.claude/agents/team/` — agent definitions as markdown files with frontmatter. Action-oriented descriptions ("Use when...", "Use proactively for...") enable automatic delegation.

### Meta-Agent Pattern

`.claude/agents/meta-agent.md` — an agent that generates new agent definition files. Outputs markdown with frontmatter to `.claude/agents/<name>.md`. Meta-skill, meta-prompt, meta-agent — skills that build skills, prompts that build prompts.

### Agentic UI Testing

User stories as simple markdown files — name, URL, workflow steps. One agent per story, all running headless Playwright in parallel. Screenshots at each step as audit trail. Best for smoke testing and visual verification where test configuration overhead exceeds benefit.

---

## 6. Distribution

**Dream state mapping:** primitive whiteboard (shared cross-org library), skills-library-mcp

### Library Meta-Skill

A YAML reference file pointing to GitHub repos (private or public) or local file paths containing skills, agents, and prompts. Managed by a single meta-skill with commands: add, use, push, sync, list, search.

**Key design:**
- **References, not copies** — the library YAML stores pointers to repos, not the actual code. `sync` pulls latest from source repos.
- **Pure agentic application** — no code, just a skill + YAML file. The agent clones repos, copies files to the right locations, pushes updates back.
- **Three scopes:** local (project `.claude/`), global (`~/.claude/`), custom (any path).
- **Push back to source:** after modifying a skill locally, `library push` commits and pushes the change back to the source repo so all consumers get the update on next sync.

**Workflow:** build (in the value-generating repo) → catalog (`library add` updates YAML references) → distribute (`library use` pulls into target codebase/device) → sync (`library sync` keeps everything current).

**Why not MCP/plugins:** private skills shouldn't be public. Teams need to control their own distribution. The YAML reference pattern is simpler than running an MCP server and works across any agent harness (Claude Code, Pi, Gemini CLI, Codex).

---

## Reference Repos

| Repo | Covers |
|------|--------|
| github.com/disler/claude-code-hooks-mastery | Validation, orchestration, agent team patterns |
| github.com/disler/claude-code-hooks-multi-agent-observability | Observability dashboard |
| github.com/disler/agent-sandbox-skill | E2B sandbox management |
| github.com/coleam00/claude-memory-compiler | Project-level knowledge accumulation |
| (mac-mini-agent, linked from video) | Dedicated agent device |
| (library, linked from video) | Distribution meta-skill |
