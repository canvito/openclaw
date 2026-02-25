# OpenClaw Agent Architecture — Learning Notes

## How OpenClaw Agents Are Configured (User Perspective)

### The Big Picture

An OpenClaw agent's personality, knowledge, and capabilities are defined almost entirely through **markdown files** and one **JSON config**. No code needed.

### 1. Workspace Files (the agent's "brain")

These live in your **workspace** (default: `~/.openclaw/workspace/`):

```
~/.openclaw/workspace/
├── AGENTS.md        ← Operating instructions ("how I work")
├── SOUL.md          ← Persona, tone, boundaries ("who I am")
├── USER.md          ← Info about you ("who you are")
├── IDENTITY.md      ← Name, emoji, avatar
├── TOOLS.md         ← Tool usage notes
├── MEMORY.md        ← Curated long-term memories
├── memory/
│   └── 2026-02-24.md  ← Daily memory logs
└── skills/
    └── my-custom-skill/
        └── SKILL.md    ← A custom skill definition
```

**Every session**, the agent reads `AGENTS.md`, `SOUL.md`, `USER.md`, and `IDENTITY.md`. These are injected directly into the system prompt. Edit them and the agent's behavior changes immediately.

### 2. Skills = Capabilities (`.md` files)

A **skill** is a folder with a `SKILL.md` file. It teaches the agent how to do something — use GitHub, check weather, run Gemini, etc. The format:

```markdown
---
name: github
description: Interact with GitHub repos, issues, and PRs
metadata: {"openclaw": {"emoji": "🐙", "requires": {"bins": ["gh"]}}}
---

## Usage
When the user asks about GitHub repos, use the `gh` CLI...
[instructions for the agent follow]
```

The **frontmatter** declares the skill's name, description, and requirements. The **body** is plain markdown instructions injected into the agent's prompt when the skill is eligible.

### 3. Three Layers of Skills (with precedence)

| Layer | Location | Wins conflicts? |
|-------|----------|----------------|
| **Workspace** | `<workspace>/skills/` | ✅ Highest |
| **Managed** | `~/.openclaw/skills/` | Middle |
| **Bundled** | Shipped with install (~40+ skills) | Lowest |

Same-named skills: workspace overrides managed, which overrides bundled. You can also add extra dirs via config.

### 4. Gating — Skills Auto-Enable/Disable

Skills self-declare what they need via `metadata.openclaw.requires`:
- **`bins`**: required CLI tools on PATH (e.g., `["gh"]`)
- **`env`**: required env vars (e.g., `["GEMINI_API_KEY"]`)
- **`config`**: required openclaw.json keys (e.g., `["browser.enabled"]`)

If requirements aren't met → skill is silently excluded from the prompt. Install the tool or set the key → skill appears automatically.

### 5. The JSON Config (`~/.openclaw/openclaw.json`)

This is where you wire up channels, models, and skill overrides:

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
  skills: {
    entries: {
      "github": { enabled: true },
      "slack": { enabled: false },
      "gemini": { apiKey: "your-key-here" },
    },
    allowBundled: ["github", "weather"],  // optional allowlist
    load: { extraDirs: ["~/my-skills"] }, // extra skill folders
  },
}
```

### 6. Memory — Short-term & Long-term

| Type | How it works |
|------|-------------|
| **Short-term** | Conversation messages kept in-session. When the context window fills up, older messages are automatically **summarized** (compacted) while preserving recent turns. |
| **Long-term** | `MEMORY.md` + `memory/*.md` files + past session transcripts are **indexed** into a SQLite vector DB. The agent does hybrid search (keyword + semantic) at the start of each turn to recall relevant knowledge. Older memories decay over time (30-day half-life). |
| **Sessions** | Stored as `.jsonl` files in `~/.openclaw/agent/*/sessions/`. Past sessions become searchable long-term memory. |

### 7. How It All Comes Together

When a message arrives, OpenClaw:

1. **Loads workspace files** → `AGENTS.md`, `SOUL.md`, `USER.md`, `IDENTITY.md` become the system prompt foundation
2. **Discovers eligible skills** → scans bundled + managed + workspace, filters by requirements → injects skill instructions into prompt
3. **Searches long-term memory** → hybrid vector+keyword search over `MEMORY.md`, session history, knowledge files → injects relevant context
4. **Registers tools** → based on eligible skills and config
5. **Runs the agent loop** → LLM generates responses + tool calls → tools execute → results fed back → repeat until done
6. **Saves to session** → conversation persisted as `.jsonl`, indexed for future memory recall

**TL;DR**: You shape the agent by editing markdown files. `SOUL.md` = personality. `AGENTS.md` = operating rules. `skills/*/SKILL.md` = capabilities. `MEMORY.md` = knowledge. `openclaw.json` = wiring. No code required.

---

## Under the Hood (Code Architecture)

### Agent Implementation & Core Loop

The agent lives in **`src/agents/`**. The core execution flow is:

```
Message arrives → agent-runner.ts → runEmbeddedPiAgent() → LLM streaming → tool execution → reply
```

**Key files:**
- **`src/agents/pi-embedded-runner/run.ts`** — Main agent loop with retry/failover logic
- **`src/agents/pi-embedded-runner/run/attempt.ts`** — Single attempt handler (init session → call LLM)
- **`src/auto-reply/reply/agent-runner.ts`** — High-level orchestrator that wires messaging, turn management, and reply delivery
- **`src/agents/pi-embedded-subscribe.ts`** — Streaming event handler that processes LLM output, dispatches tool calls, and emits reply blocks

The agent loop: **build prompt → call LLM (streaming) → parse tool calls → execute tools (with policy checks) → feed results back → repeat until done → deliver reply**.

### Prompt Organization

Prompts are **modular and section-based**, assembled in **`src/agents/system-prompt.ts`** via `buildAgentSystemPrompt()`.

**Sections assembled dynamically:**
- **Identity** — agent persona (`src/agents/identity.ts`)
- **Skills** — loaded from bundled + workspace + managed + plugin sources (`src/agents/skills/workspace.ts`), max 150 skills / 30KB
- **Memory** — recall instructions
- **Messaging** — routing, subagent orchestration
- **Tooling** — tool narration guidelines + summaries (`src/agents/tool-summaries.ts`)
- **Workspace** — paths, guidance
- **Runtime** — agent ID, host, OS, model, channel, thinking level
- **Safety** — constraints

**Prompt modes:** `"full"` (main agent), `"minimal"` (subagents), `"none"` (bare identity).

**Tools** are registered in **`src/agents/pi-tools.ts`** — defines core tools (read, write, exec, browser, message, etc.), applies policy filtering, and handles schema compatibility across providers (Claude, OpenAI, Gemini).

### Orchestration

The orchestration is a **streaming tool-use loop**:

1. **`agent-runner.ts`** receives a message, loads session context, builds prompt + tools
2. **`runEmbeddedPiAgent()`** calls the LLM via `pi-agent-core` (Anthropic's Pi library)
3. **`pi-embedded-subscribe.ts`** subscribes to the stream and dispatches events:
   - Text blocks → streamed to channel
   - Tool calls → validated by **`pi-tools.policy.ts`** → executed by **`pi-embedded-subscribe.handlers.ts`**
   - Tool results → fed back to LLM for next turn
4. **Subagents** can be spawned via `src/agents/tools/agent-step.ts` for delegation
5. **Retries/failover** handled at the `run.ts` level

### Memory Management (Code Level)

**Short-term (conversation context):**
- In-memory message arrays during a session
- History limiting (`src/agents/pi-embedded-runner/history.ts`) — keeps last N turns (configurable per-user/provider)
- Context compaction (`src/agents/compaction.ts`) — when the context window fills up, older messages are LLM-summarized into 1–2 summary blocks while preserving decisions, TODOs, and constraints. Recent turns are never removed.
- Token estimation with a 1.2x safety margin; base chunk ratio ~40%

**Long-term (persistent knowledge):**
- **`src/memory/manager.ts`** — core memory manager using **SQLite + sqlite-vec** (vector extension) + **FTS5** (full-text search)
- **Hybrid search** (`src/memory/hybrid.ts`) — combines BM25 keyword search + vector similarity (semantic)
- **Embeddings** (`src/memory/embeddings.ts`) — multi-provider: OpenAI, Gemini, Voyage, Mistral, local LLaMA
- **Temporal decay** (`src/memory/temporal-decay.ts`) — exponential score decay based on memory age (default half-life: 30 days)
- **Query expansion** (`src/memory/query-expansion.ts`) — automatic keyword extraction for better retrieval

**Session persistence:**
- Transcripts saved as **JSONL** in `~/.openclaw/agent/*/sessions/*.jsonl`
- Sessions are **dual-purpose**: conversation logs AND indexed knowledge sources for long-term memory
- On `/new` or `/reset`, a session hook generates an LLM-summarized slug to capture context
