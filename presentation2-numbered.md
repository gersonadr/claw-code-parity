# Presentation Transcript — Numbered Sentences

1. Hey everyone, let's talk about agentic AI.

2. We've seen a team prompted a microservice from scratch in a single session.

3. Another team automated a migration that would otherwise take weeks.

4. So I started reading about Harness — skills, workflows, agents — hoping to understand the magic behind it.

5. But I felt like I was missing something fundamental.

6. I didn't fully understand how these agentic AI tools work.

7. Like, how does the AI decide to start an agent?
   - ✅ **Confirmed** — `Agent` is a registered `ToolSpec` sent to the LLM on every message. The LLM invokes it the same way it invokes any other tool. (`rust/crates/tools/src/lib.rs:571`)

8. Better yet, what even is an agent?
   - ✅ **Confirmed** — Structurally an `AgentJob`: a bundle of prompt, type-specific system prompt, and type-specific allowed tool set, running in its own named OS thread. (`rust/crates/tools/src/lib.rs`)

9. Do agents communicate with each other?
   - ✅ **Confirmed** — Via a file-based manifest system. Each background agent writes `{id}.json` (status/lane events) and `{id}.md` (output) under `.clawd-agents/`. (`rust/crates/tools/src/lib.rs:3275`)

10. What's the difference between an Explorer agent and a Planner agent?
    - ✅ **Confirmed** — Two differences: (1) allowed tool set — Explore is read-only, Plan adds `TodoWrite` but no bash; (2) system prompt — the type name is injected directly into the prompt. (`rust/crates/tools/src/lib.rs:3187`)

11. Do they share the same context window?
    - ✅ **Confirmed — they do not share it.** Each agent gets `Session::new()` (empty history, zero tokens) and its own `UsageTracker` instance. Nothing is shared across threads. (`rust/crates/runtime/src/session.rs:134`, `rust/crates/runtime/src/usage.rs:169`)

12. Then in the same week Claude Code gets leaked.

13. You probably heard about it on the news.

14. So what if instead of reading these markdown files and making assumptions on how these things work, we focus on understanding the tool first?

15. This is what this talk is.

16. I promise 2 things if you pay attention.

17. One: you will build a mental model of how agentic AI tools work under the hood.

18. And two: these insights will serve you when you start building your own harness.

19. Cool.

20. In the middle we have an LLM, this is OpenAI model which you access using the API key or Ollama running on your local.
    - ❌ **Inaccurate for this codebase** — The default and primary provider is Anthropic/Claude (`https://api.anthropic.com`, `ANTHROPIC_API_KEY`). OpenAI is supported as an optional alternative. Ollama has no support at all. The speaker was using this as a generic illustrative example across the ecosystem. (`rust/crates/api/src/providers/anthropic.rs:21`)

21. On the top we have our tool set which comes pre-installed when you install, say, Codex, open code, etc.

22. On the bottom we have the session aka message history, which is an append only JSONL format, and on the right side is the context window.

23. These 3, the tool set, the context window and session are the state which your CLI tool maintains on your local environment.

24. So let's say I open Claude Code, and type in "write a unit test".

25. The CLI tool sends a message in a certain format.

26. Please notice how the message has a list of available tools, and I can show the actual message, 1s, you can see that we describe the tools for the LLM to use, also the params, and notice this Agent tool which we'll discuss in a second.

27. From now on, I'll refer to the Agentic CLI as client and the LLM as server.

28. The client writes the message to the session / aka message history, and sends the entire message history to the LLM.

29. The LLM will then start a long lived HTTP connection called SSE, or server sent events, and the client will handle the events coming from the server.

30. So, message start, message delta, and so on, and the client updates the Terminal UI.

31. Once it receives a message stop, it saves it in the session.
    - ✅ **Confirmed** — On `MessageStop`, the client assembles the full `AssistantMessage` from buffered events and calls `session.push_message()`. (`rust/crates/runtime/src/conversation.rs:361`)

32. Also, please note that the message stop contained output and input tokens, so the client updates the context window count.
    - ✅ **Confirmed** (minor precision: token counts arrive in `message_delta`, the event immediately before `message_stop`). `UsageTracker.record()` accumulates input and output tokens. (`rust/crates/runtime/src/usage.rs:192`)

33. Now the client knows that it should read a file, so it executes the read file tool and adds the output of the read file tool to the session, and again, sends the entire session to the LLM.
    - ✅ **Confirmed** — The main `loop {}` executes tools, appends results to the session, then restarts — building a new `ApiRequest` from the full session each time. (`rust/crates/runtime/src/conversation.rs:312`)

34. Again, LLM will process, sends the response to the client, this time to write a file.
    - ✅ **Confirmed** — Same loop, next iteration.

35. The client executes the tool, appends the result, and calls the LLM again.
    - ✅ **Confirmed** — Same loop.

36. And so the process continues, until something interesting happens.
    - ✅ **Confirmed** — `loop {}` with no fixed iteration cap for the foreground agent.

37. This time, the LLM asked to use the Agent tool.
    - ✅ **Confirmed** — `"Agent"` is matched and dispatched in the tool handler. (`rust/crates/tools/src/lib.rs:1196`)

38. What happens?

39. The client will then create a new thread, this thread will then communicate with the LLM.
    - ✅ **Confirmed** — `spawn_agent_job()` calls `std::thread::Builder::new().name(...).spawn()`, creating a named OS thread that runs its own `ConversationRuntime` loop. (`rust/crates/tools/src/lib.rs:3106`)

40. It's a set & forget pattern.
    - ✅ **Confirmed** — The thread handle is immediately discarded via `.map(|_| ())`; there is no `.join()` call. The foreground receives the manifest with `status: "running"` and moves on. (`rust/crates/tools/src/lib.rs:3128`)

41. There are a few differences between the background agent and the foreground agent.

42. Let's say foreground is the agent the user interacts with.

43. First, the background agent doesn't have Agent tool in its allowed list.

44. So this answers the question of loop detection: the client structurally enforces agents up to depth 1.

45. Another difference is this max iteration count: technically the foreground agent is also limited by Integer MAX_VALUE, meaning, infinite iterations, but the background is supposed to be short lived, so, it enforces a hard stop at 32.

46. Also, notice the background agent has a clean session, with only one text task, and this task is the exact prompt the LLM informed previously.

47. So the background agent sends the session to the LLM, and LLM responds with another tool use, so the loop continues.

48. One key insight for me that you might have already realized: all communications with the LLM are stateless.

49. It doesn't know if it's talking to a foreground or a background agent.

50. Similarly, the agent itself doesn't know what they are, and I find this design to be elegant.

51. So, this time the background agent runs the bash tool, which executes the newly created test class.
    - ✅ **Confirmed** — `"bash"` is explicitly listed in the general-purpose (default) agent's allowed tool set. (`rust/crates/tools/src/lib.rs:3244`)

52. Background agent adds the result to the session, and sends again to the LLM, and so on.
    - ✅ **Confirmed** — `run_agent_job()` uses the same `ConversationRuntime` loop as the foreground; tool results are appended and the LLM is called again identically. (`rust/crates/tools/src/lib.rs:3133`)

53. In the meantime, the foreground agent is constantly polling a manifest file, which is a json file with the agent id.
    - ❌ **Inaccurate** — There is no polling loop in the Rust code. The foreground fires and forgets. The manifest JSON file (`{id}.json`) is real and on disk, but the foreground never reads it. If status tracking is needed, it is the LLM itself that would have to call `read_file` on the manifest path in a loop. (`rust/crates/tools/src/lib.rs:3097`)

54. Once the status = "complete" the foreground agent reads the background agents' output from the markdown file.
    - ⚠️ **Partially inaccurate** — The output file is correctly a `.md` file, but the status string in code is `"completed"`, not `"complete"`. (`rust/crates/tools/src/lib.rs:3139`)

55. I have a small example.

56. This answered another question I had about how agents communicate.

57. For this implementation it is a simple poll, but other tools, such as Opencode, they use http response.
    - ❌ **Inaccurate** — There is no poll. Communication is purely file-based and one-directional: the background thread writes to disk, nothing reads it back on the foreground side. (`rust/crates/tools/src/lib.rs:3097`)

58. But that's just implementation details.

59. Now notice what happened on the right side: the context window counter has reached a critical threshold, and now the client will perform a compact operation.
    - ✅ **Confirmed** — Threshold is 100,000 cumulative input tokens. When reached, compaction is triggered inside the conversation loop. (`rust/crates/runtime/src/conversation.rs:18`)

60. This operation takes all the messages (excluding the 4 most recent ones) and summarizes into one.
    - ✅ **Confirmed** — `preserve_recent_messages: 4` is the exact value in `CompactionConfig`. (`rust/crates/runtime/src/compact.rs:18`)

61. This summary message has a list of files read, written, list of prompts by the user and the LLM, but all truncated at 160 characters.

62. So it's a lossy compression.

63. While I was reading I was curious as to why the client doesn't submit the message queue for the LLM to compress, and the reason is quite simple: it doesn't want to spend too many tokens specially because the context is almost full.

64. Plus, this is a blocking operation on this Conversation loop, it must be fast, and let's not forget the LLM still has the full context of the last 4 messages, so it knows what to do.

65. The client sends the entire session again, and the LLM returns with only text, meaning, no tool use, and the exit criteria for this loop is met.

66. So right now the client is waiting for the prompt by the user.

67. So this is the pseudo code of what we just discussed.

68. Start an infinite loop, increment interactions, call LLM, listen to events, build the AssistantMessage struct from the events, save on the session, if no tool use, break, else execute the tool.

69. What I didn't cover is that this client supports pre tool hook and post tool hook.

70. So say you want audit logs of Jenkins tools, and after the job is done, send a Slack message.

71. Or maybe you want to add another layer of safety, and check if the client will remove a file it's not supposed to, so the pre hook task replaces the tool with echo "Denied" so the LLM knows it's not allowed.
    - ✅ **Confirmed in spirit** — A pre-hook's stdout becomes the denial message returned to the LLM. The hook sets `is_denied()` which produces `PermissionOutcome::Deny`. The `echo "Denied"` example is accurate conceptually. (`rust/crates/runtime/src/conversation.rs:371`, `rust/crates/runtime/src/hooks.rs:452`)

72. Then we also have the permission_policy which is as you would expect.
    - ✅ **Confirmed** — `PermissionPolicy` is a first-class struct stored in `ConversationRuntime` and consulted at every tool authorization. (`rust/crates/runtime/src/permissions.rs:98`)

73. One point about the permission policy: if you are a foreground agent, you have access to the Prompter, which lets you ask questions to the user.
    - ✅ **Confirmed** — The `PermissionPrompter` trait is the Prompter. Foreground calls `run_turn(input, Some(&mut permission_prompter))`; background calls `run_turn(prompt, None)`. (`rust/crates/rusty-claude-cli/src/main.rs:2421`)

74. So if the tool policy is askUser, it may proceed.
    - ⚠️ **Partially inaccurate** — The concept is correct but the value is not named `askUser`. The equivalent in code is `PermissionMode::Prompt` (triggers user prompt) and `PermissionOverride::Ask` (hook-level override). (`rust/crates/runtime/src/permissions.rs:8`)

75. But the background agent doesn't have the prompter object, so they are always auto-approved or auto-denied, given it's a background process that should not be interrupted.
    - ✅ **Confirmed** — Background passes `None` for the prompter. Policy is built with `DangerFullAccess` as the base mode, and tool access is pre-filtered by `allowed_tools_for_subagent()`. All decisions are static with no user interaction. (`rust/crates/tools/src/lib.rs:3136`)

76. I still had one question left unanswered, which is what is the difference between an Explorer agent and a Planner agent.

77. There are 2 main differences: allowed tools and system prompt.
    - ✅ **Confirmed** — Each type maps to a distinct `BTreeSet` in `allowed_tools_for_subagent()` and its type name is injected into the system prompt via `build_agent_system_prompt()`. (`rust/crates/tools/src/lib.rs:3187`)

78. Here we have some examples for a few clients, notice how the nomenclature varies but both have a general agent and an explore agent, who can only do read-only tasks.
    - ✅ **Confirmed for this codebase** — The Explore agent's tool set contains only read-only tools; `bash`, `write_file`, and `edit_file` are absent. Cross-tool comparison with other clients is not verifiable here. (`rust/crates/tools/src/lib.rs:3187`)

79. So I had another question then: how does the LLM know which agent the client is able to start?

80. Does the client also send a list of supported agents?
    - ✅ **Confirmed — it does not.** `subagent_type` is a plain `"type": "string"` in the schema with no `enum` constraint. No list of valid types is sent to the LLM anywhere. (`rust/crates/tools/src/lib.rs:578`)

81. The answer for this implementation is no: the LLM does provide an agent type on the message payload, but it's a free text.
    - ✅ **Confirmed** — Whatever string the LLM writes for `subagent_type` is accepted; normalisation happens entirely client-side. (`rust/crates/tools/src/lib.rs:570`)

82. The client then normalizes this value and maps to their list of available agent types.
    - ✅ **Confirmed** — `normalize_subagent_type()` is called immediately after parsing the Agent tool input, before the job is built. (`rust/crates/tools/src/lib.rs:3043`)

83. If you can imagine a switch statement where the first clause is "does it contain plan", etc, and the default clause, aka fallback agent, is the general purpose.
    - ✅ **Confirmed in structure** — It is a Rust `match` (equivalent to switch) with a `"plan" | "planagent"` arm. Minor note: matching is on exact canonicalised tokens, not substrings. Also, truly unknown types pass through as-is rather than defaulting to general-purpose; the general-purpose fallback applies only when input is empty. (`rust/crates/tools/src/lib.rs:3863`)

84. This is not all bad, but as best practice it is always better to limit the tooling and scope of your agents to the task to avoid inconsistencies.

85. If I have some time, I can briefly talk about env isolation (another thing I was wondering about), and the Fork() operation which explains how some agentic tools support restoring to a particular snapshot.
    - ✅ **Confirmed** — `fork_managed_session()` exists and copies the full session message history into a new session with a fresh ID, recording `parent_session_id` and an optional `branch_name`. Persisted to disk, enabling snapshot/restore. (`rust/crates/runtime/src/session_control.rs:273`)

86. Else, if anyone has any questions, please feel free.
