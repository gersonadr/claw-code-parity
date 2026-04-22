# Source Code Verification — Sentences 81–86

---

## Sentences 84, 86 — Not verifiable

| # | Sentence | Verdict |
|---|---|---|
| 84 | "As best practice it is always better to limit the tooling and scope of your agents to the task to avoid inconsistencies." | N/A — best practice advice |
| 86 | "Else, if anyone has any questions, please feel free." | N/A — closing remark |

---

## Sentence 81 — "The LLM does provide an agent type on the message payload, but it's a free text."

**Verdict: CONFIRMED**

Already established in the S80 verification: the `subagent_type` field in the Agent tool's JSON schema is typed as a plain `"type": "string"` with no `enum` constraint. Whatever string the LLM writes is accepted as-is and only normalised client-side.

**`rust/crates/tools/src/lib.rs` lines 570–584:**
```rust
"subagent_type": { "type": "string" }   // no enum, no allowed values list
```

---

## Sentence 82 — "The client then normalizes this value and maps to their list of available agent types."

**Verdict: CONFIRMED**

`normalize_subagent_type()` is called immediately after the Agent tool input is parsed, before the agent job is built.

**`rust/crates/tools/src/lib.rs` line 3043:**
```rust
let normalized_subagent_type = normalize_subagent_type(input.subagent_type.as_deref());
```

The function passes the raw string through `canonical_tool_token()` (strips whitespace, lowercases, removes non-alphanumeric characters) and then maps it to the internal type name.

---

## Sentence 83 — "A switch statement where the first clause is 'does it contain plan', and the default clause is the general purpose."

**Verdict: CONFIRMED in structure — minor precision note on matching**

The normalization is a Rust `match` (the direct equivalent of a switch statement). The speaker's description is accurate — "plan" is one of the clauses and the default falls back to general-purpose.

**`rust/crates/tools/src/lib.rs` lines 3857–3874:**
```rust
fn normalize_subagent_type(subagent_type: Option<&str>) -> String {
    let trimmed = subagent_type.map(str::trim).unwrap_or_default();
    if trimmed.is_empty() {
        return String::from("general-purpose");   // ← empty input → default
    }

    match canonical_tool_token(trimmed).as_str() {
        "general" | "generalpurpose" | "generalpurposeagent" => String::from("general-purpose"),
        "explore" | "explorer"  | "exploreagent"             => String::from("Explore"),
        "plan"    | "planagent"                              => String::from("Plan"),   // ← plan clause
        "verification" | "verificationagent" | "verify" | "verifier" => String::from("Verification"),
        "clawguide" | "clawguideagent" | "guide"             => String::from("claw-guide"),
        "statusline" | "statuslinesetup"                     => String::from("statusline-setup"),
        _ => trimmed.to_string(),   // ← unknown type passed through as-is
    }
}
```

**Precision note:** The speaker said "does it *contain* plan" suggesting a substring check, but the code performs exact matching on the canonicalised token (`"plan"` or `"planagent"`). The spirit is the same — any reasonable "plan" variant maps to the Plan agent. Also note: truly unknown types fall through to `trimmed.to_string()` rather than to general-purpose; the general-purpose fallback applies only when the input is empty or blank.

---

## Sentence 85 — "The Fork() operation explains how some agentic tools support restoring to a particular snapshot."

**Verdict: CONFIRMED — Fork() exists in this codebase**

`fork_managed_session()` and `session.fork()` are real functions. Forking copies the current session's message history into a new session with its own ID while recording `parent_session_id` and an optional `branch_name` — creating a named snapshot that can be resumed independently.

**`rust/crates/runtime/src/session_control.rs` lines 273–297:**
```rust
pub fn fork_managed_session(
    session: &Session,
    branch_name: Option<String>,
) -> Result<ForkedManagedSession, SessionControlError> {
    fork_managed_session_for(env::current_dir()?, session, branch_name)
}

pub fn fork_managed_session_for(
    base_dir: impl AsRef<Path>,
    session: &Session,
    branch_name: Option<String>,
) -> Result<ForkedManagedSession, SessionControlError> {
    let parent_session_id = session.session_id.clone();
    let forked = session.fork(branch_name);             // copy messages, new session_id
    let handle = create_managed_session_handle_for(base_dir, &forked.session_id)?;
    forked.save_to_path(&handle.path)?;                 // persist to disk
    Ok(ForkedManagedSession {
        parent_session_id,                              // lineage preserved
        handle,
        session: forked,
    })
}
```

**`rust/crates/runtime/src/conversation.rs` line 508** — runtime-level fork entry point:
```rust
pub fn fork_session(&self, branch_name: Option<String>) -> Session {
    self.session.fork(branch_name)
}
```

**Test confirming lineage — `rust/crates/runtime/src/session_control.rs` line 446:**
```rust
assert_eq!(forked.parent_session_id, source.session_id);
assert_eq!(forked.branch_name.as_deref(), Some("incident-review"));
```

A fork preserves the full message history at the point of forking, records the parent, and saves to a new file — enabling a user to restore or branch off from any prior state.

---

## Summary

| # | Sentence | Verdict |
|---|---|---|
| 81 | LLM provides agent type as free text | **Confirmed** — `subagent_type` is a plain string in the schema |
| 82 | Client normalises and maps to known types | **Confirmed** — `normalize_subagent_type()` called at line 3043 |
| 83 | Switch statement with "plan" clause and general-purpose default | **Confirmed in structure** — exact `match` with `"plan" \| "planagent"` arm; default for empty input is general-purpose; unknown strings pass through rather than defaulting |
| 84 | Best practice advice | N/A |
| 85 | Fork() operation enables snapshot/restore | **Confirmed** — `fork_managed_session()` copies session history with parent lineage into a new persisted session |
| 86 | Closing remark | N/A |
