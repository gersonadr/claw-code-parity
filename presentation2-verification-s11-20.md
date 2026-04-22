# Source Code Verification — Sentences 11–20

---

## Sentences 12–19 — Not verifiable in source code

These are rhetorical, anecdotal, or news-event statements with no corresponding technical claim to check.

| # | Sentence | Verdict |
|---|---|---|
| 12 | "Then in the same week Claude Code gets leaked." | N/A — news event |
| 13 | "You probably heard about it on the news." | N/A — anecdotal |
| 14 | "So what if instead of reading these markdown files and making assumptions…" | N/A — rhetorical |
| 15 | "This is what this talk is." | N/A — rhetorical |
| 16 | "I promise 2 things if you pay attention." | N/A — rhetorical |
| 17 | "One: you will build a mental model of how agentic AI tools work under the hood." | N/A — rhetorical |
| 18 | "And two: these insights will serve you when you start building your own harness." | N/A — rhetorical |
| 19 | "Cool." | N/A — filler |

---

## Sentence 11 — "Do they share the same context window?"

**Verdict: CONFIRMED — they do NOT share it. Each agent has its own independent context window.**

Background agents are spawned with a brand-new `Session` and a brand-new `UsageTracker`. No counter is passed in or shared across threads.

**Fresh session per background agent — `rust/crates/tools/src/lib.rs` lines 3155–3161:**
```rust
Ok(ConversationRuntime::new(
    Session::new(),   // clean slate — zero messages, zero tokens
    api_client,
    tool_executor,
    permission_policy,
    job.system_prompt.clone(),
))
```

**`Session::new()` initialises an empty message vector — `rust/crates/runtime/src/session.rs` lines 134–145:**
```rust
pub fn new() -> Self {
    Self {
        version:      SESSION_VERSION,
        session_id:   generate_session_id(),
        created_at_ms: now,
        updated_at_ms: now,
        messages:     Vec::new(),   // empty — no shared history
        compaction:   None,
        fork:         None,
        persistence:  None,
    }
}
```

**Independent token counter per runtime — `rust/crates/runtime/src/conversation.rs` line 174:**
```rust
let usage_tracker = UsageTracker::from_session(&session);
```

**`UsageTracker` struct — `rust/crates/runtime/src/usage.rs` lines 169–189:**
```rust
pub struct UsageTracker {
    latest_turn:  TokenUsage,
    cumulative:   TokenUsage,
    turns:        u32,
}
```

`UsageTracker` is stored as a private field inside `ConversationRuntime` and is never wrapped in `Arc<Mutex<>>` or otherwise shared. Each agent thread owns its own instance.

---

## Sentence 20 — "In the middle we have an LLM, this is OpenAI model which you access using the API key or Ollama running on your local."

**Verdict: INACCURATE — the primary and default LLM is Anthropic Claude, not OpenAI. Ollama is not supported at all.**

### What the code actually shows

**Default API endpoint — `rust/crates/api/src/providers/anthropic.rs` line 21:**
```rust
pub const DEFAULT_BASE_URL: &str = "https://api.anthropic.com";
```

**Authentication env vars — `rust/crates/api/src/providers/anthropic.rs` lines 40–55:**
```
ANTHROPIC_API_KEY   (primary)
ANTHROPIC_AUTH_TOKEN (alternative)
ANTHROPIC_BASE_URL   (optional override)
```

**Default model names — `rust/crates/api/src/providers/mod.rs` lines 126–129:**
```
claude-sonnet-4-6
claude-opus-4-6
claude-haiku-4-5-20251213
```

**Model alias mapping — `rust/crates/api/src/providers/mod.rs` lines 147–153:**
Anything starting with `"claude"` is routed to the Anthropic provider automatically.

### OpenAI status

OpenAI is supported as an **optional alternative** via `OpenAiCompatClient` and the `OPENAI_API_KEY` env var (`rust/crates/api/src/providers/openai_compat.rs` lines 48–54). It is not the default.

### Ollama status

No references to Ollama, local model hosting, or local endpoints exist anywhere in the provider implementations.

### Why the statement is understandable

The speaker was presenting a **general conceptual model** of how agentic CLI tools work across the ecosystem (Codex, OpenCode, etc.), not describing this specific codebase. The architecture diagram they described — LLM in the middle, tool set on top, session on the bottom — is accurate for all of them. The "OpenAI / Ollama" example was illustrative, not a claim about this implementation.

---

## Summary

| # | Sentence | Verifiable? | Verdict |
|---|---|---|---|
| 11 | Do they share the same context window? | Yes | **Confirmed** — each agent gets `Session::new()` and its own `UsageTracker`; no sharing |
| 12–19 | Opening / rhetorical | No | N/A |
| 20 | "OpenAI model or Ollama running on your local" | Yes | **Inaccurate** for this codebase — default provider is Anthropic/Claude; OpenAI is an optional alternative; Ollama is not supported |
