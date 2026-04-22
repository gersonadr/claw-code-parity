# Source Code Verification — Sentences 31–40

---

## Sentence 38 — Not verifiable

| # | Sentence | Verdict |
|---|---|---|
| 38 | "What happens?" | N/A — rhetorical question |

---

## Sentence 31 — "Once it receives a message stop, it saves it in the session."

**Verdict: CONFIRMED**

The `MessageStop` SSE event sets a `finished` flag, at which point the client assembles the full `AssistantMessage` from all buffered events and calls `session.push_message()`.

**`rust/crates/runtime/src/conversation.rs` lines 693–694** — stop event marks stream as finished:
```rust
AssistantEvent::MessageStop => {
    finished = true;
}
```

**`rust/crates/runtime/src/conversation.rs` lines 333–340** — message is assembled from streamed events:
```rust
let (assistant_message, usage, turn_prompt_cache_events) =
    match build_assistant_message(events) {
        Ok(result) => result,
        Err(error) => {
            self.record_turn_failed(iterations, &error);
            return Err(error);
        }
    };
```

**`rust/crates/runtime/src/conversation.rs` lines 361–363** — assembled message is saved to the session:
```rust
self.session
    .push_message(assistant_message.clone())
    .map_err(|error| RuntimeError::new(error.to_string()))?;
```

---

## Sentence 32 — "The message stop contained output and input tokens, so the client updates the context window count."

**Verdict: CONFIRMED** (with a minor precision note: token counts arrive in the `message_delta` event that immediately precedes `message_stop`, not in `message_stop` itself)

**`rust/crates/api/src/sse.rs` lines 161–164** — SSE stream order shows token usage in `message_delta`:
```
event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"tool_use"},"usage":{"input_tokens":1,"output_tokens":2}}

event: message_stop
```

**`rust/crates/tools/src/lib.rs` lines 3489–3494** — both events are handled back-to-back:
```rust
ApiStreamEvent::MessageDelta(delta) => {
    events.push(AssistantEvent::Usage(delta.usage.token_usage()));
},
ApiStreamEvent::MessageStop(_) => {
    saw_stop = true;
    events.push(AssistantEvent::MessageStop);
},
```

**`rust/crates/runtime/src/conversation.rs` lines 341–342** — usage is recorded into the tracker:
```rust
if let Some(usage) = usage {
    self.usage_tracker.record(usage);
}
```

**`rust/crates/runtime/src/usage.rs` lines 192–198** — `record()` accumulates input and output tokens:
```rust
pub fn record(&mut self, usage: TokenUsage) {
    self.latest_turn = usage;
    self.cumulative.input_tokens  += usage.input_tokens;
    self.cumulative.output_tokens += usage.output_tokens;
    self.cumulative.cache_creation_input_tokens += usage.cache_creation_input_tokens;
    self.cumulative.cache_read_input_tokens     += usage.cache_read_input_tokens;
    self.turns += 1;
}
```

---

## Sentences 33–35 — Tool execution loop

**33:** "The client executes the read file tool and adds the output to the session, and again, sends the entire session to the LLM."
**34:** "LLM will process, sends the response to the client, this time to write a file."
**35:** "The client executes the tool, appends the result, and calls the LLM again."

**Verdict: CONFIRMED**

All three sentences describe the same loop. The main conversation loop in `conversation.rs` continuously: sends the full session to the LLM → receives a response → saves the assistant message → executes any tool calls → appends results → loops back to the top.

**`rust/crates/runtime/src/conversation.rs` — main loop structure (lines 312–470):**

```
loop {                                          // line 312 — restarts here each turn
    │
    ├─ Build ApiRequest from full session        // line 322–325
    ├─ Send to LLM (SSE stream)                 // line 326
    ├─ Build AssistantMessage from events        // line 333–340
    ├─ Record token usage                        // line 341–342
    ├─ Save AssistantMessage to session          // line 361–363
    ├─ If no tool calls → break                 // line 366–369
    │
    └─ For each tool call:
        ├─ Execute tool                          // line 370–460
        ├─ Append ToolResult to session          // line 461–466
        └─ (loop restarts)
}
```

After tool results are appended the loop restarts at line 312, and the new `ApiRequest` at line 322 is built from the full session — which now includes all prior messages plus the new tool results.

---

## Sentences 36–37 — "The process continues until the LLM uses the Agent tool."

**36:** "And so the process continues, until something interesting happens."
**37:** "This time, the LLM asked to use the Agent tool."

**Verdict: CONFIRMED**

Sentence 36 describes the loop continuing, which is confirmed by the `loop {}` structure above. Sentence 37 is confirmed: `"Agent"` is a registered tool the LLM can choose to invoke at any iteration, dispatched at `rust/crates/tools/src/lib.rs` line 1196:
```rust
"Agent" => from_value::<AgentInput>(input).and_then(run_agent)
```

---

## Sentence 39 — "The client will then create a new thread, this thread will then communicate with the LLM."

**Verdict: CONFIRMED**

When the Agent tool is invoked, `spawn_agent_job()` creates a named OS thread that runs its own independent conversation loop with the LLM.

**`rust/crates/tools/src/lib.rs` lines 3106–3130:**
```rust
fn spawn_agent_job(job: AgentJob) -> Result<(), String> {
    let thread_name = format!("clawd-agent-{}", job.manifest.agent_id);
    std::thread::Builder::new()
        .name(thread_name)
        .spawn(move || {
            let result = std::panic::catch_unwind(
                std::panic::AssertUnwindSafe(|| run_agent_job(&job))
            );
            match result {
                Ok(Ok(())) => {}
                Ok(Err(error)) => {
                    let _ = persist_agent_terminal_state(
                        &job.manifest, "failed", None, Some(error)
                    );
                }
                Err(_) => {
                    let _ = persist_agent_terminal_state(
                        &job.manifest, "failed", None,
                        Some(String::from("sub-agent thread panicked")),
                    );
                }
            }
        })
        .map(|_| ())
        .map_err(|error| error.to_string())
}
```

`run_agent_job()` inside the thread runs the same `ConversationRuntime` loop as the foreground agent.

---

## Sentence 40 — "It's a set & forget pattern."

**Verdict: CONFIRMED**

The thread handle returned by `std::thread::spawn` is immediately discarded via `.map(|_| ())` — there is no `.join()` call anywhere in `spawn_agent_job`. The foreground agent receives the manifest (with `status: "running"`) and continues immediately without waiting for the background thread to finish.

**`rust/crates/tools/src/lib.rs` lines 3128–3130:**
```rust
        })
        .map(|_| ())          // handle dropped — no join, no wait
        .map_err(|error| error.to_string())
}
```

**`rust/crates/tools/src/lib.rs` lines 3103–3104** — caller returns the manifest immediately:
```rust
    Ok(manifest)   // status = "running"; background thread still executing
}
```

---

## Summary

| # | Sentence | Verdict |
|---|---|---|
| 31 | On `message_stop`, saves to session | **Confirmed** — `push_message()` called after assembling events |
| 32 | `message_stop` contains tokens, client updates context window count | **Confirmed** (tokens arrive in `message_delta` immediately before stop; `UsageTracker.record()` accumulates them) |
| 33 | Executes tool, appends result, sends full session to LLM | **Confirmed** — main `loop {}` in `conversation.rs` |
| 34 | LLM responds with another tool call | **Confirmed** — same loop, next iteration |
| 35 | Client executes, appends, calls LLM again | **Confirmed** — same loop |
| 36 | Process continues | **Confirmed** — `loop {}` with no fixed iteration cap for foreground |
| 37 | LLM invokes the Agent tool | **Confirmed** — `"Agent"` dispatched at line 1196 |
| 38 | "What happens?" | N/A — rhetorical |
| 39 | Client creates a new thread | **Confirmed** — `std::thread::Builder::new().spawn()` in `spawn_agent_job()` |
| 40 | Set & forget pattern | **Confirmed** — thread handle discarded with `.map(|_| ())`; no `.join()` |
