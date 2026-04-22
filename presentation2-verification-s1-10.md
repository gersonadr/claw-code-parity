# Source Code Verification — Sentences 1–10

---

## Sentences 1–6 — Not verifiable in source code

These are anecdotal and motivational statements (personal experience, observations about teams, feelings of confusion). No source code can confirm or deny them.

| # | Sentence | Verdict |
|---|---|---|
| 1 | "Hey everyone, let's talk about agentic AI." | N/A — opening remark |
| 2 | "We've seen a team prompted a microservice from scratch in a single session." | N/A — anecdotal |
| 3 | "Another team automated a migration that would otherwise take weeks." | N/A — anecdotal |
| 4 | "So I started reading about Harness — skills, workflows, agents — hoping to understand the magic behind it." | N/A — personal motivation |
| 5 | "But I felt like I was missing something fundamental." | N/A — personal feeling |
| 6 | "I didn't fully understand how these agentic AI tools work." | N/A — personal feeling |

---

## Sentence 7 — "How does the AI decide to start an agent?"

**Verdict: CONFIRMED**

The `Agent` is a registered tool presented to the LLM in every message. The LLM decides to invoke it the same way it decides to invoke any other tool — by including it in its response as a tool call. There is no special logic; the LLM sees the tool and chooses it.

**`rust/crates/tools/src/lib.rs` lines 571–584:**
```rust
ToolSpec {
    name: "Agent",
    description: "Launch a specialized agent task and persist its handoff metadata.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "description": { "type": "string" },
            "prompt":      { "type": "string" },
            "subagent_type": { "type": "string" },
            "name":        { "type": "string" },
            "model":       { "type": "string" }
        },
        "required": ["description", "prompt"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::DangerFullAccess,
},
```

When the LLM returns a call to `"Agent"`, the client dispatches it (line 1196):
```rust
"Agent" => from_value::<AgentInput>(input).and_then(run_agent)
```

---

## Sentence 8 — "What even is an agent?"

**Verdict: CONFIRMED**

An agent is structurally an `AgentJob`, a bundle containing a manifest, a prompt, a type-specific system prompt, and a type-specific allowed tool set. It runs in its own OS thread.

**`rust/crates/tools/src/lib.rs` — key structs:**

```rust
// The input the LLM provides when invoking the Agent tool
struct AgentInput {
    description:   String,
    prompt:        String,
    subagent_type: Option<String>,
    name:          Option<String>,
    model:         Option<String>,
}

// The internal job passed to the worker thread
struct AgentJob {
    manifest:      AgentOutput,   // identity + status + file paths
    prompt:        String,
    system_prompt: Vec<String>,   // type-specific instructions
    allowed_tools: BTreeSet<String>, // type-specific tool list
}
```

The job is spawned as a named OS thread (line 3107):
```rust
fn spawn_agent_job(job: AgentJob) -> Result<(), String> {
    let thread_name = format!("clawd-agent-{}", job.manifest.agent_id);
    std::thread::Builder::new()
        .name(thread_name)
        .spawn(move || { run_agent_job(&job) })
}
```

---

## Sentence 9 — "Do agents communicate with each other?"

**Verdict: CONFIRMED — via manifest files on disk**

Agents do not call each other directly. They communicate through a file-based manifest system: each background agent writes its state and output to two files under `.clawd-agents/`:

| File | Contents |
|---|---|
| `{agent_id}.json` | JSON manifest (`AgentOutput`) — status, timestamps, lane events |
| `{agent_id}.md` | Text output of the completed agent |

**`rust/crates/tools/src/lib.rs` — manifest write (lines 3275–3281):**
```rust
fn write_agent_manifest(manifest: &AgentOutput) -> Result<(), String> {
    std::fs::write(
        &manifest.manifest_file,
        serde_json::to_string_pretty(manifest).map_err(|e| e.to_string())?,
    )
    .map_err(|e| e.to_string())
}
```

The `AgentOutput` struct tracks `status`, `output_file`, `manifest_file`, `lane_events`, and `current_blocker` — everything the foreground agent needs to know what the background agent did.

---

## Sentence 10 — "What's the difference between an Explorer agent and a Planner agent?"

**Verdict: CONFIRMED — two differences: allowed tools and system prompt**

The presentation states exactly two differences. The code confirms both.

### Difference 1: Allowed tools

**`rust/crates/tools/src/lib.rs` lines 3187–3263:**

| Agent type | Can read files | Can run bash | Can write files | Can use TodoWrite | Can spawn agents |
|---|---|---|---|---|---|
| **Explore** | Yes | No | No | No | No |
| **Plan** | Yes | No | No | Yes | No |
| **general-purpose** (default) | Yes | Yes | Yes | Yes | No |

Explore is strictly read-only. Plan adds `TodoWrite` and `SendUserMessage` but still no bash.

### Difference 2: System prompt

**`rust/crates/tools/src/lib.rs` lines 3164–3178:**
```rust
fn build_agent_system_prompt(subagent_type: &str) -> Result<Vec<String>, String> {
    let mut prompt = load_system_prompt(...)?;
    prompt.push(format!(
        "You are a background sub-agent of type `{subagent_type}`. \
         Work only on the delegated task, use only the tools available to you, \
         do not ask the user questions, and finish with a concise result."
    ));
    Ok(prompt)
}
```

The `subagent_type` string is embedded directly into the system prompt, so an Explore agent and a Plan agent receive different instructions that shape their behaviour.

### How agent types are resolved from the LLM's free-text value

**`rust/crates/tools/src/lib.rs` lines 3857–3875:**
```rust
fn normalize_subagent_type(subagent_type: Option<&str>) -> String {
    match canonical_tool_token(trimmed).as_str() {
        "general" | "generalpurpose" | "generalpurposeagent" => "general-purpose",
        "explore"  | "explorer"  | "exploreagent"            => "Explore",
        "plan"     | "planagent"                             => "Plan",
        "verification" | "verificationagent" | "verify"      => "Verification",
        "clawguide" | "guide"                                => "claw-guide",
        "statusline" | "statuslinesetup"                     => "statusline-setup",
        _ => trimmed,   // unknown type passed through as-is → fallback to general-purpose
    }
}
```

The LLM provides a free-text agent type; the client normalises it and picks the matching tool set and system prompt. The default fallback is `general-purpose`.

---

## Summary

| # | Sentence | Verifiable? | Verdict |
|---|---|---|---|
| 1–6 | Opening / anecdotal | No | N/A |
| 7 | How does the AI decide to start an agent? | Yes | **Confirmed** — `Agent` is a registered `ToolSpec`; LLM invokes it like any other tool |
| 8 | What even is an agent? | Yes | **Confirmed** — `AgentJob` struct with prompt, system prompt, allowed tools; runs in its own thread |
| 9 | Do agents communicate with each other? | Yes | **Confirmed** — file-based manifest system (`{id}.json` + `{id}.md`) under `.clawd-agents/` |
| 10 | Explorer vs Planner difference? | Yes | **Confirmed** — differ in (1) allowed tool set and (2) system prompt |
