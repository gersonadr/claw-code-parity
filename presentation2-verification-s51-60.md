# Source Code Verification — Sentences 51–60

---

## Sentences 55, 56, 58 — Not verifiable

| # | Sentence | Verdict |
|---|---|---|
| 55 | "I have a small example." | N/A — live demo reference |
| 56 | "This answered another question I had about how agents communicate." | N/A — rhetorical |
| 58 | "But that's just implementation details." | N/A — rhetorical |

---

## Sentence 51 — "The background agent runs the bash tool, which executes the newly created test class."

**Verdict: CONFIRMED**

The default/general-purpose background agent explicitly includes `"bash"` in its allowed tool set.

**`rust/crates/tools/src/lib.rs` lines 3244–3263** — default case in `allowed_tools_for_subagent()`:
```rust
_ => BTreeSet::from([
    "bash",          // ← bash is allowed
    "read_file",
    "write_file",
    "edit_file",
    "glob_search",
    "grep_search",
    "WebFetch",
    "WebSearch",
    "TodoWrite",
    "Skill",
    "ToolSearch",
    // ...
]),
```

The fallback agent type is `"general-purpose"` (line 3860), which hits this default case.

---

## Sentence 52 — "Background agent adds the result to the session, and sends again to the LLM, and so on."

**Verdict: CONFIRMED**

`run_agent_job()` constructs a `ConversationRuntime` — the exact same runtime used by the foreground agent — and calls `run_turn()` on it. The runtime's internal `loop {}` handles appending tool results to the session and re-sending to the LLM identically for both agents.

**`rust/crates/tools/src/lib.rs` lines 3133–3136:**
```rust
fn run_agent_job(job: &AgentJob) -> Result<(), String> {
    let mut runtime = build_agent_runtime(job)?;
    runtime.run_turn(job.prompt.clone(), None)
        .map_err(|error| error.to_string())?;
    // ...
}
```

**`rust/crates/tools/src/lib.rs` lines 3155–3161** — runtime is built with a fresh session:
```rust
Ok(ConversationRuntime::new(
    Session::new(),
    api_client,
    tool_executor,
    permission_policy,
    job.system_prompt.clone(),
))
```

`run_turn()` enters the same `loop {}` in `conversation.rs` that executes tools, appends results to the session, and calls the LLM again — same as the foreground.

---

## Sentence 53 — "The foreground agent is constantly polling a manifest file, which is a JSON file with the agent id."

**Verdict: INACCURATE — there is no polling**

After spawning the background thread, the foreground returns immediately with the manifest and moves on. No polling loop, no `sleep`, no periodic status check exists in the foreground for the background agent's execution.

**`rust/crates/tools/src/lib.rs` lines 3097–3104** — foreground returns right after spawn:
```rust
if let Err(error) = spawn_fn(job) {
    // handle spawn failure
    return Err(error);
}

Ok(manifest)   // ← returns immediately; status is still "running"
```

The manifest is a real JSON file on disk (`{agent_id}.json`) and the output is a real `.md` file — the file-based communication mechanism is accurate — but the foreground does not poll them. It is a true fire-and-forget.

---

## Sentence 54 — "Once the status = 'complete' the foreground agent reads the background agents' output from the markdown file."

**Verdict: PARTIALLY INACCURATE**

- The output file is indeed a `.md` file ✅
- The status string is `"completed"`, not `"complete"` ✗

**`rust/crates/tools/src/lib.rs` line 3139** — terminal state is written as `"completed"`:
```rust
persist_agent_terminal_state(&job.manifest, "completed", ...)
```

**`rust/crates/tools/src/lib.rs` line 3041** — output file path uses `.md` extension:
```rust
output_dir.join(format!("{agent_id}.md"))
```

**Test assertion confirming both — `rust/crates/tools/src/lib.rs` line 5855:**
```rust
assert!(completed_manifest.contains("\"status\": \"completed\""));
// line 5854:
std::fs::read_to_string(&completed.output_file)  // reads the .md file
```

As noted in sentence 53, the foreground does not actively read the output file after completion — it fires and forgets. The manifest and output file are written by the background thread for the LLM / user to reference later.

---

## Sentence 57 — "For this implementation it is a simple poll, but other tools, such as Opencode, they use http response."

**Verdict: INACCURATE — it is not a poll; it is fire-and-forget**

As established above, there is no polling in this implementation. The foreground spawns the thread and immediately returns the manifest. Communication happens entirely through files written by the background thread. Calling it a "simple poll" overstates the foreground's involvement.

The contrast with Opencode's HTTP response approach is a fair observation about the ecosystem, but it cannot be verified in this codebase.

---

## Sentence 59 — "The context window counter has reached a critical threshold, and now the client will perform a compact operation."

**Verdict: CONFIRMED**

There is an explicit token threshold constant. When cumulative input tokens reach or exceed it, compaction is triggered.

**`rust/crates/runtime/src/conversation.rs` line 18** — threshold constant:
```rust
const DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD: u32 = 100_000;
```

**`rust/crates/runtime/src/conversation.rs` line 519** — threshold check in the loop:
```rust
if self.usage_tracker.cumulative_usage().input_tokens
    < self.auto_compaction_input_tokens_threshold
{
    // skip compaction
} else {
    // trigger compaction
}
```

When cumulative input tokens hit 100,000 the client enters the compaction path.

---

## Sentence 60 — "This operation takes all the messages (excluding the 4 most recent ones) and summarizes into one."

**Verdict: CONFIRMED — the number 4 is exact**

**`rust/crates/runtime/src/compact.rs` line 18** — default config:
```rust
preserve_recent_messages: 4,
```

**`rust/crates/runtime/src/compact.rs` lines 111–115** — compaction logic preserves the tail:
```rust
// All messages except the last `preserve_recent_messages` are
// replaced with a single summary message.
```

The 4 most recent messages are kept verbatim; everything older is collapsed into one summary message.

---

## Summary

| # | Sentence | Verdict |
|---|---|---|
| 51 | Background agent runs bash tool | **Confirmed** — `"bash"` is in the general-purpose agent's allowed tool set |
| 52 | Background agent appends results, re-sends to LLM | **Confirmed** — `run_agent_job()` uses the same `ConversationRuntime` loop |
| 53 | Foreground constantly polls a manifest JSON file | **Inaccurate** — foreground fires and forgets; no polling loop exists |
| 54 | Status = "complete", output read from markdown file | **Partially inaccurate** — status string is `"completed"` (not `"complete"`); `.md` output file is correct |
| 55 | "I have a small example." | N/A |
| 56 | "This answered another question…" | N/A |
| 57 | Simple poll (vs Opencode's HTTP) | **Inaccurate** — not a poll; fire-and-forget via manifest files |
| 58 | "That's just implementation details." | N/A |
| 59 | Threshold triggers compact operation | **Confirmed** — 100,000 cumulative input tokens triggers compaction |
| 60 | All messages except 4 most recent summarized into one | **Confirmed** — `preserve_recent_messages: 4` in `CompactionConfig` |
