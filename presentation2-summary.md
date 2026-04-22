# Agentic AI Under the Hood — Main Points

## Context / Motivation

- Real teams prompted microservices from scratch and automated multi-week migrations in a single session.
- Reading framework docs (Harness, skills, workflows, agents) left fundamental questions unanswered.
- Goal: build a mental model of how agentic AI tools actually work, then apply it when building your own harness.

---

## Core Architecture

Four components, always present:

| Component | What it is |
|---|---|
| **LLM** | The "server" — OpenAI API, Ollama, etc. Stateless; knows nothing about who is calling it. |
| **Tool set** | Pre-installed capabilities (read file, bash, agent, etc.) described to the LLM in every message. |
| **Session / message history** | Append-only JSONL log of every message. The entire history is sent on every call. |
| **Context window** | Token counter maintained by the CLI ("client"). |

---

## The Conversation Loop

1. User types a prompt → client formats a message **including the tool list** and appends it to the session.
2. Client sends the full session to the LLM over a **long-lived SSE (Server-Sent Events) connection**.
3. Client streams events (`message_start`, `message_delta`, `message_stop`) and updates the terminal UI.
4. On `message_stop`, the client saves the response, updates the token count, and checks for tool use.
5. If the LLM returned a tool call → client executes it, appends the result, and loops back to step 2.
6. If the LLM returned only text (no tool use) → loop exits, client waits for the next user prompt.

---

## Agents (Background vs. Foreground)

When the LLM invokes the **Agent tool**, the client spawns a new thread — a background agent.

| Property | Foreground agent | Background agent |
|---|---|---|
| Agent tool available | Yes | **No** (prevents recursion deeper than depth 1) |
| Max iterations | Effectively unlimited (Integer.MAX_VALUE) | Hard cap of **32** |
| Session | Carries full history | Starts **clean** — only the task prompt |
| User interaction | Has a `Prompter` (can ask questions) | **No Prompter** — auto-approve or auto-deny |

Key insight: **all LLM calls are stateless**. The LLM never knows if it is talking to a foreground or background agent — the client enforces the constraints structurally.

---

## Agent Communication

- The foreground agent polls a **manifest JSON file** that contains the background agent's ID and status.
- When `status == "complete"`, the foreground agent reads the background agent's output from a markdown file.
- Other tools (e.g., Opencode) use HTTP responses instead — implementation detail, same pattern.

---

## Context Window Compaction

When the token count approaches a critical threshold:

- The client takes **all messages except the 4 most recent** and summarizes them locally (no LLM call).
- The summary includes: files read/written, user prompts, LLM responses — all **truncated at 160 characters**.
- This is **lossy compression** done client-side to avoid spending tokens when the window is nearly full.
- The LLM retains full context of the last 4 messages, so it can continue without interruption.

---

## Hooks

- **Pre-tool hook**: runs before a tool executes. Use case: safety check — replace a destructive tool call with `echo "Denied"` so the LLM knows it was blocked.
- **Post-tool hook**: runs after a tool executes. Use case: audit logging, Slack notifications on job completion.

---

## Permission Policy

- Foreground agents have a `Prompter` object — if policy is `askUser`, the user is prompted interactively.
- Background agents have no `Prompter` — tools are **auto-approved or auto-denied**; background processes must not be interrupted.

---

## Agent Types (Explorer vs. Planner)

Two differences between agent types:

1. **Allowed tools** — e.g., an Explorer agent is restricted to read-only tools.
2. **System prompt** — shapes the agent's behavior and scope.

The LLM specifies an agent type as **free text** in the message payload. The client normalizes this string (think: a switch statement checking for "plan", "explore", etc.) and maps it to a known type. The **default / fallback is the general-purpose agent**.

Best practice: limit the tooling and scope of each agent type to avoid inconsistencies.

---

## Pseudocode — Conversation Loop

```
loop forever:
    iterations++
    response = call_llm(session)
    assistant_message = build_from_events(response)
    session.append(assistant_message)
    if no tool_use in assistant_message:
        break
    result = execute_tool(assistant_message.tool_use)
    session.append(result)
```
