# Source Code Verification — Sentences 71–80

---

## Sentences 76, 79 — Not verifiable

| # | Sentence | Verdict |
|---|---|---|
| 76 | "I still had one question left unanswered, which is what is the difference between an Explorer agent and a Planner agent." | N/A — rhetorical (already verified in s1-10 doc) |
| 79 | "So I had another question then: how does the LLM know which agent the client is able to start?" | N/A — rhetorical |

---

## Sentence 71 — "The pre hook task replaces the tool with echo 'Denied' so the LLM knows it's not allowed."

**Verdict: CONFIRMED in spirit — mechanism is slightly different**

A pre-hook can block a tool and its stdout becomes the denial message returned to the LLM. There is no literal `echo "Denied"` string hardcoded, but a hook script that outputs anything and exits with the deny code achieves exactly this effect.

**`rust/crates/runtime/src/conversation.rs` lines 371–400** — pre-hook runs before every tool, result decides fate:
```rust
let pre_hook_result = run_pre_tool_use_hook(...);

// hook can rewrite the input
let effective_input = pre_hook_result.updated_input()
    .unwrap_or(input);

if pre_hook_result.is_denied() {
    PermissionOutcome::Deny { reason: pre_hook_result.message() }
}
```

**`rust/crates/runtime/src/hooks.rs` line 452** — default deny message format:
```rust
"{} hook denied tool `{tool_name}`"
```

**`rust/crates/runtime/src/hooks.rs` line 88** — hooks can also rewrite the tool input before execution:
```rust
updated_input: Option<String>,
```

So a pre-hook can either block the tool entirely (deny) or rewrite its input. The LLM receives the hook's stdout as the tool result, which can say anything — including "Denied".

---

## Sentence 72 — "Then we also have the permission_policy which is as you would expect."

**Verdict: CONFIRMED**

`PermissionPolicy` is a first-class struct used throughout the runtime.

**`rust/crates/runtime/src/permissions.rs` lines 98–105:**
```rust
pub struct PermissionPolicy {
    active_mode:       PermissionMode,
    tool_requirements: BTreeMap<String, PermissionMode>,
    allow_rules:       Vec<...>,
    deny_rules:        Vec<...>,
    ask_rules:         Vec<...>,
}
```

It is stored as a field in `ConversationRuntime` (`conversation.rs` line 130) and called at every tool authorization check (`conversation.rs` lines 402–414).

---

## Sentence 73 — "If you are a foreground agent, you have access to the Prompter, which lets you ask questions to the user."

**Verdict: CONFIRMED**

The `PermissionPrompter` trait is the Prompter. The foreground passes a live implementation; the background passes `None`.

**`rust/crates/runtime/src/permissions.rs` lines 86–88** — trait definition:
```rust
pub trait PermissionPrompter {
    fn decide(&mut self, request: &PermissionRequest) -> PermissionPromptDecision;
}
```

**`rust/crates/rusty-claude-cli/src/main.rs` lines 2420–2421** — foreground wires in a real prompter:
```rust
let mut permission_prompter = CliPermissionPrompter::new(self.permission_mode);
let result = runtime.run_turn(input, Some(&mut permission_prompter));
//                                    ^^^^ prompter present
```

**`rust/crates/tools/src/lib.rs` line 3136** — background passes nothing:
```rust
let summary = runtime.run_turn(job.prompt.clone(), None);
//                                                  ^^^^ no prompter
```

---

## Sentence 74 — "So if the tool policy is askUser, it may proceed."

**Verdict: PARTIALLY INACCURATE — the value is called `Prompt`, not `askUser`**

There is no `askUser` variant. The equivalent in code is `PermissionMode::Prompt`, which triggers the user prompt flow. Hooks can also return `PermissionOverride::Ask` to request user input.

**`rust/crates/runtime/src/permissions.rs` lines 8–14** — actual `PermissionMode` enum variants:
```rust
pub enum PermissionMode {
    ReadOnly,
    WorkspaceWrite,
    DangerFullAccess,
    Prompt,            // ← this is the "ask user" mode
    Allow,
}
```

**`rust/crates/runtime/src/permissions.rs` lines 31–36** — hook-level override:
```rust
pub enum PermissionOverride {
    Ask,    // ← hooks can request user input
    Allow,
    Deny,
}
```

The concept is correct — there is a policy value that routes to user prompting — but the name in code is `Prompt` / `Ask`, not `askUser`.

---

## Sentence 75 — "The background agent doesn't have the prompter object, so they are always auto-approved or auto-denied."

**Verdict: CONFIRMED**

Background agents receive `None` for the prompter (confirmed in sentence 73 evidence). Their permission policy is constructed with `DangerFullAccess` as the base mode, and tool access is pre-filtered by `allowed_tools_for_subagent()` — tools not in the allowed set are auto-denied structurally, those in it are auto-approved.

**`rust/crates/tools/src/lib.rs` lines 3268–3272:**
```rust
fn agent_permission_policy() -> PermissionPolicy {
    // base mode: DangerFullAccess — no interactive prompts
    ...
}
```

**`rust/crates/tools/src/lib.rs` lines 3187–3266** — allowed tools list per type enforces auto-deny for anything outside it.

When `prompter` is `None`, `conversation.rs` lines 401–415 follow static rules only — no user interaction possible.

---

## Sentence 77 — "There are 2 main differences: allowed tools and system prompt."

**Verdict: CONFIRMED** (previously verified in s1-10 doc — restated here for completeness)

Each agent type maps to a distinct `BTreeSet` of allowed tools (`allowed_tools_for_subagent()`, lines 3187–3263) and receives its type name injected into the system prompt (`build_agent_system_prompt()`, lines 3164–3177).

---

## Sentence 78 — "Both have a general agent and an explore agent, who can only do read-only tasks."

**Verdict: CONFIRMED for this codebase — cross-tool comparison is not verifiable here**

The Explorer agent's allowed tool set contains only read-only tools (no `bash`, `write_file`, `edit_file`):

**`rust/crates/tools/src/lib.rs` lines 3187–3199:**
```rust
"Explore" => BTreeSet::from([
    "read_file",
    "glob_search",
    "grep_search",
    "WebFetch",
    "WebSearch",
    "ToolSearch",
    "Skill",
    "StructuredOutput",
    // no bash, no write_file, no edit_file
]),
```

The claim that other tools (Opencode, Codex, etc.) follow the same pattern cannot be verified from this codebase.

---

## Sentence 80 — "Does the client also send a list of supported agents?"

**Verdict: CONFIRMED — it does NOT send a list**

The `subagent_type` field in the Agent tool's JSON schema is a plain `"type": "string"` with no `enum` constraint. No list of valid types is enumerated anywhere in the tool description, system prompt, or API message.

**`rust/crates/tools/src/lib.rs` lines 570–584** — Agent tool schema:
```rust
ToolSpec {
    name: "Agent",
    description: "Launch a specialized agent task and persist its handoff metadata.",
    input_schema: json!({
        "properties": {
            "subagent_type": { "type": "string" },  // ← free text, no enum
            ...
        }
    }),
}
```

The recognised types (`Explore`, `Plan`, `Verification`, etc.) live only in `normalize_subagent_type()` (lines 3857–3874) and are never sent to the LLM.

---

## Summary

| # | Sentence | Verdict |
|---|---|---|
| 71 | Pre-hook replaces tool with "Denied" | **Confirmed in spirit** — hook stdout becomes the denial message; mechanism matches, exact wording is implementation detail |
| 72 | `permission_policy` exists | **Confirmed** — `PermissionPolicy` struct in `permissions.rs` |
| 73 | Foreground has Prompter, background does not | **Confirmed** — `Some(&mut prompter)` foreground vs `None` background in `run_turn()` |
| 74 | Tool policy value is `askUser` | **Partially inaccurate** — concept is correct but the actual value is `PermissionMode::Prompt` / `PermissionOverride::Ask` |
| 75 | Background always auto-approved or auto-denied | **Confirmed** — `None` prompter + `DangerFullAccess` base + pre-filtered allowed tool set |
| 76 | Rhetorical | N/A |
| 77 | Two differences: tools and system prompt | **Confirmed** (previously verified) |
| 78 | Explore agent is read-only | **Confirmed** for this codebase — cross-tool comparison unverifiable |
| 79 | Rhetorical | N/A |
| 80 | Client does NOT send a list of agent types | **Confirmed** — `subagent_type` is a free-text string in the schema; no enum sent to LLM |
