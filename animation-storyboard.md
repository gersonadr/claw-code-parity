# Animation Storyboard — Claude Conversation Loop
## "The Memory Tape + Tool Shelf" — Manim Production Script

> **Use case throughout this video:**
> The user asks: *"Find every Python file in this project that imports requests,
> and write a markdown report summarising what each one does."*
>
> This single request will produce:
> - A `bash` tool call (find .py files)
> - A `grep` tool call (filter imports)
> - An `agent` tool call (sub-agent reads files & drafts summaries)
>   - Sub-agent uses: `read` (×3), `grep` (×1)
> - A `write` tool call (save the report)
> - A final text reply
>
> Total: 3 outer tool calls + 1 agent + 4 inner tool calls = 2 agent-loop iterations

---

## 1. Canvas & Coordinate System

Manim default frame: **14 units wide × 8 units tall**, origin at center.

```
X axis:  -7.0 (left edge)  →  +7.0 (right edge)
Y axis:  -4.0 (bottom)     →  +4.0 (top)
```

### Zone Map

```
Y = +4.0 ┌──────────────────────────────────────────────┐
          │           TOOL SHELF  (Y = +3.0)            │
Y = +2.0  │                                              │
          │       CLAUDE BRAIN  center (0, +0.6)        │
Y =  0.0  │                                              │
          │  TOKEN METER (right, X=+6.2)                 │
Y = -2.0  │                                              │
          │══════════ MEMORY TAPE  (Y = -3.2) ══════════│
Y = -4.0  └──────────────────────────────────────────────┘
```

---

## 2. Visual Element Definitions

### 2.1 Memory Tape

| Property | Value |
|---|---|
| Shape | Rectangle strip, full width |
| Position | Center X = 0, Y = -3.2 |
| Size | Width grows rightward; height = 0.55 units |
| Background | Dark gray `#1a1a2e`, subtle border `#444` |
| Blocks snap onto | Right end of tape; tape auto-scrolls left when too wide |

**Tape Block types:**

| Type | Color | Label style |
|---|---|---|
| User message | `#3a86ff` (blue) | white monospace, truncated to ~20 chars |
| Assistant message | `#9b5de5` (purple) | white monospace |
| Tool result (success) | `#06d6a0` (green) | tool icon + short label |
| Tool result (error) | `#ef233c` (red) | tool icon + "error" |
| Summary (compacted) | `#6c757d` (gray) | "∑ summary" label |

Block dimensions: width = proportional to content (min 1.0, max 2.2 units), height = 0.45 units.
Blocks have 0.08 unit gap between them. Corners rounded at radius 0.06.

---

### 2.2 Claude Brain

| Property | Value |
|---|---|
| Shape | Circle |
| Position | (0, +0.6) |
| Radius | 0.9 units |
| Color (idle) | `#9b5de5` with 30% opacity fill, `#9b5de5` stroke |
| Color (thinking) | Pulsing glow — stroke brightens, fill opacity rises to 70% |
| Label | "Claude" in white, size 0.28, centered |
| Pulse period | 1.2 s sinusoidal scale oscillation (0.9 → 1.05 → 0.9 radius) |

---

### 2.3 Tool Shelf

The shelf is a horizontal row of tool objects at Y = +3.0. Tools are evenly spaced.
Shelf background: semi-transparent dark bar across full width, height 0.9 units, Y = +3.0.

**5 tools, X positions:**

| Tool | X pos | Shape | Icon color (idle) | Icon color (active) |
|---|---|---|---|---|
| `bash` | -4.8 | Rounded rect (terminal) | `#40916c` dim | `#74c69d` bright |
| `grep` | -2.4 | Circle with magnifying glass | `#1d6fa4` dim | `#48cae4` bright |
| `read` | 0.0 | Open book (two rect halves) | `#b5838d` dim | `#ffb4a2` bright |
| `write` | +2.4 | Pencil (thin rotated rect + tip) | `#e07a5f` dim | `#f2cc8f` bright |
| `agent` | +4.8 | Small circle (brain, same as Claude) | `#6d3b8e` dim | `#c77dff` bright |

Each tool object:
- Outer bounding box: 1.0 × 0.7 units
- Label below icon: tool name, size 0.18, white 50% opacity when idle, 100% active
- Idle state: Y = +3.0, scale = 1.0, dim colors
- Active state: Y = +3.15 (lifted 0.15), scale = 1.1, bright colors, drop shadow

---

### 2.4 Token Meter

| Property | Value |
|---|---|
| Shape | Vertical bar, rounded ends |
| Position | X = +6.2, Y center = 0 |
| Size | 0.3 wide × 4.0 tall |
| Background | Dark `#222` |
| Fill | Gradient: green → yellow → red (bottom to top) |
| Fill level | Grows upward proportional to token count |
| Label | "tokens" rotated 90°, size 0.18, X = +6.55 |
| Threshold line | Dashed red at 85% fill — triggers compaction |

---

### 2.5 Arc / Signal Lines

When the brain sends a tool call:
- A curved arc (quadratic bezier) draws from the brain center to the tool on the shelf
- Color: matches the tool's active color
- Style: dashed, animated dash-offset so it looks like a moving signal
- Arrowhead at the tool end
- Duration: 0.5 s draw

When a result packet returns:
- A straight line with result packet (capsule shape) travels from tool down to tape
- Packet color matches tool result block color (`#06d6a0`)
- Packet has small tool icon branded on it
- Duration: 0.6 s travel, ease-in

---

### 2.6 Iteration Counter

| Property | Value |
|---|---|
| Position | X = -6.2, Y = +1.5 |
| Label | "iteration" size 0.18 + large digit size 0.55 |
| Color | White, dim between increments |
| Animation | Number flips (like a ticker) on each loop-back |

---

### 2.7 Loop-Back Arrow

A large arc arrow that lives in the background, sweeping from the right end of the tape
up and around to the brain input.

- Color: `#adb5bd` (muted gray)
- Appears only during the loop-back moment (step S13)
- Animates as a traveling dot along the arc path
- Fades out once the brain starts pulsing again

---

### 2.8 Sub-Agent Overlay (agent tool only)

When the agent tool fires, an overlay appears on top of the main scene:

| Property | Value |
|---|---|
| Background | Semi-transparent dark overlay, 70% black |
| Sub-brain | Circle at (0, +0.6), radius 0.55 (smaller than Claude) |
| Sub-brain color | `#c77dff` |
| Sub-tape | Same style as main tape but Y = -2.5, width capped at 10 units |
| Sub-shelf | Smaller row of tools, Y = +2.5, scale 0.75 |
| Border | Dashed purple rectangle framing the whole sub-scene |
| Label | "sub-agent" top-left of frame, size 0.22, `#c77dff` |

The sub-agent overlay fades in (0.4 s) when the agent tool activates and fades out (0.6 s)
when it completes, collapsing into a result packet.

---

## 3. Example Use Case — Narrative Arc

```
User:   "Find every Python file that imports requests,
         and write a markdown report of what each one does."

Turn 1 — Agent loop iteration 1:
  Claude → bash("find . -name '*.py'")           [result: 6 file paths]
  Claude → grep("import requests", files)         [result: 3 matching files]

Turn 1 — Agent loop iteration 2:
  Claude → agent("read the 3 files and summarise")[sub-agent runs its own loop]
    Sub-agent iter 1:
      sub-Claude → read("api_client.py")
      sub-Claude → read("scraper.py")
      sub-Claude → read("utils.py")
    Sub-agent iter 2:
      sub-Claude → grep("def ", all three files)
    Sub-agent → final text reply (summaries)
  Claude → write("report.md", <content>)         [result: file written]

Turn 1 — Agent loop exits (no more tool calls):
  Claude → final text reply: "Done. report.md created."
```

---

## 4. Scene-by-Scene Script

> **Timing conventions**
> - Narrator pace: ~130 words/minute (≈ 2.2 words/second)
> - Animation time listed separately from narration time
> - Scene duration = max(narration, animations) + hold
> - Transitions between scenes: 0.3 s cross-fade unless noted

---

### SCENE 1 — Stage Reveal
**Cumulative time: 0:00 – 0:12**

#### Elements entering
| Element | Animation | Duration |
|---|---|---|
| Dark background | Fade in | 0.4 s |
| Tool shelf bar | Slide down from top | 0.5 s |
| Tool icons (one by one, left→right) | Pop in with slight bounce | 0.15 s each = 0.75 s total |
| Memory tape strip | Slide up from bottom | 0.5 s |
| Token meter | Fade in, empty | 0.3 s |
| Iteration counter "0" | Fade in | 0.3 s |
| Claude brain circle | Scale from 0 → 1, ease-out | 0.6 s |

Total animation: ~3.5 s

#### Narrator text (22 words ≈ 10 s)
> *"Claude Code works through a loop — a cycle that repeats until the work is done.
> Three elements drive it: the brain, the tape, and the tools."*

#### Hold after narration: 1.5 s
**Scene total: ~12 s**

---

### SCENE 2 — User Message Arrives
**Cumulative time: 0:12 – 0:26**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| Blue user block | Slides in from left edge, snaps onto tape | 0.7 s |
| Block label text | Types in character by character (truncated) | 0.8 s |
| Tape | Nudges right slightly to accommodate block | 0.3 s |

Total animation: ~2 s

#### Narrator text (28 words ≈ 13 s)
> *"The user sends a message: 'Find every Python file that imports requests and write
> a markdown report of what each one does.' It lands on the tape as the first block."*

#### Hold: 1.0 s
**Scene total: ~14 s**

---

### SCENE 3 — Tape Streams Into Brain (API Request)
**Cumulative time: 0:26 – 0:42**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| Tape blocks | Duplicate copies lift off, stream upward toward brain as a column | 1.0 s |
| Brain | Brightens, begins slow pulse as copies arrive | 0.4 s |
| Streaming copies | Disappear as they enter brain circle | 0.5 s |
| Brain glow | Intensifies to full brightness | 0.3 s |
| Label below brain | "reading context…" fades in | 0.3 s |

Total animation: ~2.5 s

#### Narrator text (30 words ≈ 14 s)
> *"Every time Claude is about to respond, it reads the entire conversation from scratch —
> every message, every tool result. This is the full context window, flowing in."*

#### Hold: 1.5 s
**Scene total: ~16 s**

---

### SCENE 4 — Streaming Response, Tool Calls Detected
**Cumulative time: 0:42 – 1:02**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| Brain pulse | Continues for 1.0 s ("thinking") | 1.0 s |
| Text particles | Flow out from brain, drift right, coalesce into purple block on tape | 1.2 s |
| Purple block | Partially forms, fills as tokens arrive | 1.0 s |
| Tool-call dots | Two glowing dots appear on the purple block | 0.4 s |
| Dot 1 arc | Draws arc from block → bash tool (left side of shelf) | 0.5 s |
| Dot 2 arc | Draws arc from block → grep tool | 0.5 s |
| bash & grep tools | Light up and lift off shelf simultaneously | 0.4 s |
| Iteration counter | Flips from 0 → 1 | 0.3 s |

Total animation: ~6 s

#### Narrator text (38 words ≈ 17 s)
> *"Claude responds — but not with words yet. It issues two tool calls: bash, to find
> Python files, and grep, to filter for the ones that import requests. Each call
> lights up its tool on the shelf."*

#### Hold: 1.0 s
**Scene total: ~20 s**

---

### SCENE 5 — bash Tool Executes
**Cumulative time: 1:02 – 1:18**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| bash tool | Terminal screen animates: text types "find . -name '*.py'" | 0.8 s |
| Terminal output | 6 file path lines scroll into view inside the tool icon | 0.8 s |
| Checkmark | Appears on bash icon | 0.3 s |
| Result packet (green) | Flies from bash icon down to tape, snaps as green block | 0.6 s |
| Green block label | "bash ✓  6 files" | 0.3 s |
| bash tool | Dims, settles back to shelf position | 0.3 s |

Total animation: ~4 s

#### Narrator text (24 words ≈ 11 s)
> *"Bash runs, finds six Python files. The result — a list of paths — flies back
> as a green block and joins the tape."*

#### Hold: 1.5 s
**Scene total: ~16 s**

---

### SCENE 6 — grep Tool Executes
**Cumulative time: 1:18 – 1:33**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| grep tool | Magnifying glass sweeps left-right over abstract file lines | 0.9 s |
| Match highlights | 3 lines glow as magnifier passes | 0.5 s |
| Checkmark | Appears on grep icon | 0.3 s |
| Result packet | Flies to tape, snaps as green block | 0.6 s |
| Green block label | "grep ✓  3 matches" | 0.3 s |
| grep tool | Dims, settles | 0.3 s |

Total animation: ~3.5 s

#### Narrator text (22 words ≈ 10 s)
> *"Grep scans those files for the import statement. Three match. Another green block
> snaps onto the tape."*

#### Hold: 1.5 s
**Scene total: ~15 s**

---

### SCENE 7 — Loop-Back Arrow (Iteration 2)
**Cumulative time: 1:33 – 1:48**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| Loop-back arrow | Traveling dot traces arc from tape's right end, up and around to brain | 1.0 s |
| Tape blocks | All three (user, assistant, bash-result, grep-result) stream into brain again | 0.9 s |
| Brain | Pulses, thinking | 0.8 s |
| Iteration counter | Flips 1 → 2 | 0.3 s |
| Loop-back arrow | Fades out | 0.3 s |

Total animation: ~3.5 s

#### Narrator text (30 words ≈ 14 s)
> *"Now Claude reads everything again — the original request, plus both tool results.
> This is iteration two. It hasn't replied yet; it has more work to plan."*

#### Hold: 0.5 s
**Scene total: ~15 s**

---

### SCENE 8 — Agent Tool Call Initiated
**Cumulative time: 1:48 – 2:05**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| Brain | Pulses, new tokens stream out → new purple block forms | 1.2 s |
| Tool-call dot | Single dot on purple block, draws arc → agent tool | 0.5 s |
| agent tool | Glows bright purple, lifts off shelf | 0.4 s |
| agent tool | Begins to expand outward from shelf position | 0.5 s |

Total animation: ~3.5 s

#### Narrator text (32 words ≈ 15 s)
> *"This time Claude delegates. It calls the agent tool — a sub-agent — asking it to
> read the three matching files and produce a summary of each one."*

#### Hold: 1.5 s
**Scene total: ~17 s**

---

### SCENE 9 — Sub-Agent Overlay Appears
**Cumulative time: 2:05 – 2:20**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| Dark overlay | Fades in over main scene (main scene dims but remains visible) | 0.5 s |
| Dashed purple border | Draws itself around sub-scene area | 0.6 s |
| Sub-agent label | "sub-agent" fades in top-left of frame | 0.3 s |
| Sub-brain circle | Scales in at (0, +0.6), smaller, purple | 0.5 s |
| Sub-tape strip | Slides up from bottom of overlay | 0.4 s |
| Sub-shelf (read, grep, scaled 0.75) | Tools pop in | 0.4 s |
| Sub-brain label | "Claude" in smaller text | 0.2 s |

Total animation: ~3.5 s

#### Narrator text (30 words ≈ 14 s)
> *"The agent tool is not just a function call — it spawns an entirely new conversation
> loop. Same structure: a brain, a tape, a shelf. Watch it run."*

#### Hold: 1.0 s
**Scene total: ~15 s**

---

### SCENE 10 — Sub-Agent Loop: Reads 3 Files
**Cumulative time: 2:20 – 2:50**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| Sub-tape | User-block (the delegated task) slides onto sub-tape | 0.5 s |
| Sub-tape → sub-brain | Streams in, sub-brain pulses | 0.7 s |
| Sub-brain → 3 arcs | Three arcs draw to read tool simultaneously | 0.5 s |
| read tool (×3, sequenced) | Book flips open, pages animate for each file; 3 green packets return one by one to sub-tape | 2.4 s (0.8 s × 3) |
| Sub-tape | Grows: 4 blocks now (task + 3 read results) | continuous |
| Sub-iteration counter (inside overlay) | 1 → 2 mid-sequence | 0.2 s |

Total animation: ~7 s

#### Narrator text (28 words ≈ 13 s)
> *"The sub-agent reads all three files. Each read result lands on its own tape,
> just like the outer loop. The sub-agent is building its own context."*

#### Hold: 1.0 s
**Scene total: ~20 s** (animation-bound)

---

### SCENE 11 — Sub-Agent Loop: grep + Final Reply
**Cumulative time: 2:50 – 3:12**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| Sub-tape → sub-brain (iter 2) | Loop-back arrow, all blocks stream in | 0.8 s |
| Sub-brain → grep arc | Arc draws to grep tool | 0.4 s |
| grep tool | Animates, returns green packet | 0.9 s |
| Sub-brain (iter 3) | Reads everything, pulses | 0.7 s |
| Sub-brain → purple block | Streams out on sub-tape (final text reply) | 0.8 s |
| No more tool dots on block | Block settles, sub-loop ends | 0.3 s |

Total animation: ~5 s

#### Narrator text (30 words ≈ 14 s)
> *"One more grep to find all function definitions. Then, with all the context it needs,
> the sub-agent writes its final answer — the file summaries — and the loop exits."*

#### Hold: 1.0 s
**Scene total: ~22 s**

---

### SCENE 12 — Sub-Agent Collapses Into Result Packet
**Cumulative time: 3:12 – 3:28**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| Sub-tape, sub-brain, sub-shelf | All shrink and converge toward center of overlay | 1.0 s |
| Sub-scene elements | Collapse into a single green result packet (capsule) with agent icon branded on it | 0.6 s |
| Dark overlay | Fades out | 0.5 s |
| Main scene | Returns to full brightness | 0.3 s |
| Result packet | Travels from where agent tool was → tape, snaps as green block | 0.7 s |
| agent tool | Dims, settles back to shelf | 0.3 s |
| Green block label | "agent ✓  summaries" | 0.2 s |

Total animation: ~4.5 s

#### Narrator text (26 words ≈ 12 s)
> *"The entire sub-agent run collapses into a single green block — the summaries —
> and snaps onto the outer tape. The outer Claude now has its answer."*

#### Hold: 1.5 s
**Scene total: ~16 s**

---

### SCENE 13 — Loop-Back Arrow (Iteration 3) + write Tool
**Cumulative time: 3:28 – 3:52**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| Loop-back arrow | Traces arc, tape streams into brain | 0.9 s |
| Brain | Pulses | 0.7 s |
| Iteration counter | 2 → 3 | 0.2 s |
| Purple block | Streams out, one dot appears → write tool arc | 1.0 s |
| write tool | Pencil animates writing lines; text fills a virtual page | 1.0 s |
| write tool checkmark | Checkmark, green packet returns to tape | 0.6 s |
| Green block label | "write ✓  report.md" | 0.2 s |
| write tool | Dims, settles | 0.2 s |

Total animation: ~6 s

#### Narrator text (28 words ≈ 13 s)
> *"Claude loops one more time. Now it calls write — the report is saved to disk.
> The pencil animates, the file appears, and the result joins the tape."*

#### Hold: 1.0 s
**Scene total: ~24 s**

---

### SCENE 14 — Final Reply, Loop Exits
**Cumulative time: 3:52 – 4:12**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| Loop-back arrow | Final arc — tape streams into brain | 0.8 s |
| Brain | Pulses | 0.7 s |
| Purple block | Streams out — final text reply; NO tool dots appear | 1.0 s |
| Loop-back arrow | Does NOT appear — instead, a clean break line appears | 0.4 s |
| "turn complete" label | Fades in below brain, then fades out | 1.0 s |
| Brain | Dims slightly (idle) | 0.4 s |
| Iteration counter | "4" glows briefly then dims | 0.3 s |

Total animation: ~5 s

#### Narrator text (26 words ≈ 12 s)
> *"This time Claude's response has no tool calls — just text. The loop sees that
> and breaks. The turn is complete."*

#### Hold: 2.0 s
**Scene total: ~20 s**

---

### SCENE 15 — Token Meter & Compaction (Epilogue A)
**Cumulative time: 4:12 – 4:35**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| Token meter | Draws attention (zoom in briefly) | 0.5 s |
| Fill level | Animates rising to ~88% (over threshold line) | 0.8 s |
| Threshold line | Pulses red | 0.4 s |
| Tape left blocks (oldest) | Shake slightly | 0.4 s |
| Compaction | Old blocks squeeze and merge into gray summary block | 1.2 s |
| Token meter fill | Drops from 88% → 45% | 0.5 s |
| Summary block label | "∑ summary" | 0.2 s |

Total animation: ~5 s

#### Narrator text (30 words ≈ 14 s)
> *"If the tape grows too long, Claude compresses it. Older messages merge into a
> summary block, freeing up space. The tape shortens; the token meter drops."*

#### Hold: 1.5 s
**Scene total: ~23 s**

---

### SCENE 16 — Session Persisted (Epilogue B)
**Cumulative time: 4:35 – 4:50**

#### Elements
| Element | Animation | Duration |
|---|---|---|
| Tape | Slides toward a small disk icon (bottom-right) | 0.8 s |
| Disk icon | Absorbs the tape; checkmark appears | 0.5 s |
| Cursor | Appears at left edge of a new empty tape (blink) | 0.4 s |
| All tools on shelf | Return to idle dim state | 0.3 s |
| Brain | Idle pulse, waiting | continuous |

Total animation: ~2.5 s

#### Narrator text (22 words ≈ 10 s)
> *"The session is saved. The tools wait. The brain is ready. One loop done —
> and the next message can begin."*

#### Hold: 2.0 s
**Scene total: ~15 s**

---

## 5. Full Timing Summary

| Scene | Description | Start | End | Duration |
|---|---|---|---|---|
| 1 | Stage Reveal | 0:00 | 0:12 | 12 s |
| 2 | User Message Arrives | 0:12 | 0:26 | 14 s |
| 3 | Tape Streams Into Brain | 0:26 | 0:42 | 16 s |
| 4 | Streaming Response + Tool Calls | 0:42 | 1:02 | 20 s |
| 5 | bash Executes | 1:02 | 1:18 | 16 s |
| 6 | grep Executes | 1:18 | 1:33 | 15 s |
| 7 | Loop-Back Arrow (iter 2) | 1:33 | 1:48 | 15 s |
| 8 | Agent Tool Initiated | 1:48 | 2:05 | 17 s |
| 9 | Sub-Agent Overlay Appears | 2:05 | 2:20 | 15 s |
| 10 | Sub-Agent Reads 3 Files | 2:20 | 2:50 | 30 s |
| 11 | Sub-Agent grep + Final Reply | 2:50 | 3:12 | 22 s |
| 12 | Sub-Agent Collapses | 3:12 | 3:28 | 16 s |
| 13 | Loop-Back + write Tool | 3:28 | 3:52 | 24 s |
| 14 | Final Reply, Loop Exits | 3:52 | 4:12 | 20 s |
| 15 | Compaction (Epilogue A) | 4:12 | 4:35 | 23 s |
| 16 | Session Saved (Epilogue B) | 4:35 | 4:50 | 15 s |
| **Total** | | **0:00** | **4:50** | **290 s** |

---

## 6. Color Palette Reference

```python
COLORS = {
    "background":      "#0d1117",
    "tape_bg":         "#1a1a2e",
    "user_block":      "#3a86ff",
    "assistant_block": "#9b5de5",
    "tool_success":    "#06d6a0",
    "tool_error":      "#ef233c",
    "summary_block":   "#6c757d",
    "brain_fill":      "#9b5de5",
    "brain_glow":      "#c77dff",
    "agent_fill":      "#6d3b8e",
    "agent_glow":      "#c77dff",
    "bash_idle":       "#40916c",
    "bash_active":     "#74c69d",
    "grep_idle":       "#1d6fa4",
    "grep_active":     "#48cae4",
    "read_idle":       "#b5838d",
    "read_active":     "#ffb4a2",
    "write_idle":      "#e07a5f",
    "write_active":    "#f2cc8f",
    "loop_arrow":      "#adb5bd",
    "threshold_line":  "#ef233c",
    "text_primary":    "#ffffff",
    "text_secondary":  "#adb5bd",
}
```
