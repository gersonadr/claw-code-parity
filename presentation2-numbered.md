# Presentation Transcript — Numbered Sentences

1. Hey everyone, let's talk about agentic AI.
2. We've seen a team prompted a microservice from scratch in a single session.
3. Another team automated a migration that would otherwise take weeks.
4. So I started reading about Harness — skills, workflows, agents — hoping to understand the magic behind it.
5. But I felt like I was missing something fundamental.
6. I didn't fully understand how these agentic AI tools work.
7. Like, how does the AI decide to start an agent?
8. Better yet, what even is an agent?
9. Do agents communicate with each other?
10. What's the difference between an Explorer agent and a Planner agent?
11. Do they share the same context window?
12. Then in the same week Claude Code gets leaked.
13. You probably heard about it on the news.
14. So what if instead of reading these markdown files and making assumptions on how these things work, we focus on understanding the tool first?
15. This is what this talk is.
16. I promise 2 things if you pay attention.
17. One: you will build a mental model of how agentic AI tools work under the hood.
18. And two: these insights will serve you when you start building your own harness.
19. Cool.
20. In the middle we have an LLM, this is OpenAI model which you access using the API key or Ollama running on your local.
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
32. Also, please note that the message stop contained output and input tokens, so the client updates the context window count.
33. Now the client knows that it should read a file, so it executes the read file tool and adds the output of the read file tool to the session, and again, sends the entire session to the LLM.
34. Again, LLM will process, sends the response to the client, this time to write a file.
35. The client executes the tool, appends the result, and calls the LLM again.
36. And so the process continues, until something interesting happens.
37. This time, the LLM asked to use the Agent tool.
38. What happens?
39. The client will then create a new thread, this thread will then communicate with the LLM.
40. It's a set & forget pattern.
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
52. Background agent adds the result to the session, and sends again to the LLM, and so on.
53. In the meantime, the foreground agent is constantly polling a manifest file, which is a json file with the agent id.
54. Once the status = "complete" the foreground agent reads the background agents' output from the markdown file.
55. I have a small example.
56. This answered another question I had about how agents communicate.
57. For this implementation it is a simple poll, but other tools, such as Opencode, they use http response.
58. But that's just implementation details.
59. Now notice what happened on the right side: the context window counter has reached a critical threshold, and now the client will perform a compact operation.
60. This operation takes all the messages (excluding the 4 most recent ones) and summarizes into one.
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
72. Then we also have the permission_policy which is as you would expect.
73. One point about the permission policy: if you are a foreground agent, you have access to the Prompter, which lets you ask questions to the user.
74. So if the tool policy is askUser, it may proceed.
75. But the background agent doesn't have the prompter object, so they are always auto-approved or auto-denied, given it's a background process that should not be interrupted.
76. I still had one question left unanswered, which is what is the difference between an Explorer agent and a Planner agent.
77. There are 2 main differences: allowed tools and system prompt.
78. Here we have some examples for a few clients, notice how the nomenclature varies but both have a general agent and an explore agent, who can only do read-only tasks.
79. So I had another question then: how does the LLM know which agent the client is able to start?
80. Does the client also send a list of supported agents?
81. The answer for this implementation is no: the LLM does provide an agent type on the message payload, but it's a free text.
82. The client then normalizes this value and maps to their list of available agent types.
83. If you can imagine a switch statement where the first clause is "does it contain plan", etc, and the default clause, aka fallback agent, is the general purpose.
84. This is not all bad, but as best practice it is always better to limit the tooling and scope of your agents to the task to avoid inconsistencies.
85. If I have some time, I can briefly talk about env isolation (another thing I was wondering about), and the Fork() operation which explains how some agentic tools support restoring to a particular snapshot.
86. Else, if anyone has any questions, please feel free.
