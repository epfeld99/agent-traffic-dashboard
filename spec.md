# Spec: Live Agent-Traffic Dashboard

Status: draft v2. Nothing is built yet.
Date: 2026-10-31

## 1. Purpose

A browser page that shows, live, every request an agent harness (for example Claude Code) sends to the model and every reply, as seen by a proxy sitting between them. The page must stay **easy to read while the conversation keeps growing**: hundreds of requests, large tool results, a long system prompt resent on every call.

The proxy sees all the data, but the raw traffic is unreadable: raw JSON, raw SSE chunks, the full history resent on every request. The dashboard turns that into a readable feed in which every part can be opened and inspected.

### Goals
- Show **every request and every reply as they happen, newest first**.
- Break each request into its parts, **clearly apart**: system instructions (in full), tool definitions, conversation so far, and the new message.
- Show the reply: the model's text and the tool calls it asks for.
- Show **each tool call beside its result**, even though the result arrives in a later request.
- Offer statistics (tokens, cost, time, tool usage) **in addition to** the messages, not instead of them. Every message can be opened and drilled into.
- Make it easy to see **what changed since the previous request**, to **filter by tool**, and to **spot errors**.
- Format well: long text collapsible, JSON readable, the parts of a request visually separated.

### Non-goals (v1)
- Changing, blocking, or replaying traffic. The dashboard is read-only.
- Running anywhere except the user's own machine.
- Supporting every provider. The Anthropic Messages API is the target. Other providers (for example OpenAI/Codex) are out of scope for v1, but the design must not rule it out (see §11).

## 2. Context: the proxy

The dashboard assumes a **transparent local reverse proxy**: the harness is pointed at it (for example via `ANTHROPIC_BASE_URL`), and the proxy forwards every request unchanged to the model API and relays the reply back as it arrives.

Properties the spec relies on:
- It runs locally, sees full request bodies, and sees the reply stream chunk by chunk.
- It forwards headers and bodies untouched, including auth headers.
- Replies arrive uncompressed (or the proxy decompresses them before inspecting them).
- It may already log traffic in human-readable form, but it doesn't offer a structured feed. The dashboard needs one.

## 3. Architecture

```
harness ──►  proxy   ──► model API
               │
               │  (new) event tap: structured events, fire-and-forget
               ▼
        dashboard backend  ──(live push)──►  browser page
        (parse + store + serve)
```

### 3.1 Proxy changes (minimal and safe)
The proxy gets a small **tap**: when it reads a request, receives the reply status, relays each chunk, finishes, or fails, it emits a structured event (§4).

Hard requirements for the tap:
1. **It must never affect forwarding.** If the tap fails, is slow, or nobody is listening, traffic keeps flowing byte for byte with no added latency. Emit without blocking (queue or background thread), drop events on overflow, and catch every exception.
2. **No change to what the harness sees.** Headers, body, and stream timing all stay the same.
3. **Redaction happens in the proxy.** `authorization`, `x-api-key`, and cookies are truncated or removed before an event leaves the proxy.
4. Any existing logging behavior of the proxy stays as is.

### 3.2 Transport (open, see §11)
The event contract in §4 does not depend on the transport. Options:
- **A. JSONL file:** the proxy appends events to `traffic.jsonl`, and the backend tails it. Simplest, survives restarts, and doubles as persistence.
- **B. In-process:** the proxy itself serves the dashboard and pushes events over SSE. Single process, but it makes the proxy bigger.
- **C. Local socket/HTTP POST** to a separate dashboard process.

Recommendation: **A**.

### 3.3 Dashboard backend
- Reads events, reassembles each exchange, parses it (§5), computes derived data (§6), and keeps state.
- Serves the static page and a live push channel to the browser (SSE recommended, since traffic only flows server → browser).
- On browser connect, sends a snapshot (summaries of all exchanges), then incremental updates. Full request and reply bodies load on demand when a card is opened. A page refresh must not lose history.

## 4. Event contract (proxy → dashboard)

One JSON object per event. Every event carries `exchange_id` (unique per HTTP request) and `ts` (ISO-8601 with milliseconds).

| `type` | Emitted when | Extra fields |
|---|---|---|
| `request` | request body fully read | `method`, `path`, `headers` (redacted), `body` (raw string; parsed JSON if valid) |
| `response_start` | upstream status received | `status`, `reason`, `headers` |
| `response_chunk` | each chunk relayed | `data` (UTF-8 string, decoded with replacement) |
| `response_end` | stream finished | `bytes`, `duration_ms` |
| `error` | upstream unreachable or exception | `message`, `stage` (`connect` / `relay`) |

Chunks are split at arbitrary byte boundaries. SSE lines can be cut mid-line, and the backend must buffer before parsing.

## 5. Parsing (backend)

### 5.1 Classify each exchange
| Kind | How to detect | Default visibility |
|---|---|---|
| **Conversation turn** | `POST /v1/messages` belonging to the main conversation | Shown |
| **Side call** | `POST /v1/messages` outside the main conversation (for example short title or summary generation, often a smaller model, different system prompt) | Shown, dimmed, with a "side call" tag |
| **Token count** | `/v1/messages/count_tokens` | Hidden by a default filter |
| **Other** | `GET`/`HEAD`, other paths | Hidden by a default filter |

### 5.2 Request body (Anthropic Messages API)
Extract: `model`, `max_tokens`, `stream`, `thinking`, `temperature` and other parameters, `system` (string or array of blocks, including `cache_control` markers), `tools[]` (name, description, input_schema), `tool_choice`, `messages[]`, `metadata`.

Content block types: `text`, `image`, `document`, `tool_use`, `tool_result` (string or blocks, may have `is_error`), `thinking`, `redacted_thinking`. Unknown block types fall back to pretty-printed JSON. They are never dropped.

### 5.3 Reply
- **Streamed** (`text/event-stream`): reassemble from `message_start`, `content_block_start`, `content_block_delta` (`text_delta`, `input_json_delta`, `thinking_delta`, `signature_delta`), `content_block_stop`, `message_delta` (`stop_reason`, `usage`), `message_stop`, `ping`, `error`. `input_json_delta` fragments are concatenated and parsed at `content_block_stop`.
- **Non-streamed** (`application/json`): parse directly.
- **Errors**: HTTP 4xx/5xx, `error` SSE events, `overloaded_error`, rate limits, connection failures from the tap, and streams cut off without `message_stop`.

## 6. Derived data

### 6.1 Splitting a request into its parts
For each conversation-turn request, the backend computes:
- **System instructions:** the full text, exactly as sent.
- **Tool definitions:** the full list.
- **Conversation so far:** the `messages[]` prefix that matches the previous request in the same conversation.
- **New message(s):** the suffix not present in the previous request. This is usually a user message or the `tool_result` blocks answering the last reply's tool calls.

If there is no previous request, or the prefix doesn't match (history compacted, edited, or rewound), all messages count as "conversation so far", and the card shows a **"history rewritten"** badge. The new-message split falls back to "the last message".

**Grouping into conversations:** key on a session identifier if the request carries one (for example inside `metadata.user_id`). Otherwise, fall back to "same system prompt plus a shared message prefix".

### 6.2 Changes since the previous request
Compared with the previous request in the same conversation:
- System instructions: unchanged, or changed (with a text diff).
- Tools: unchanged, or added/removed names and changed schemas.
- Messages: N new messages, or history rewritten.
- Parameters: model, max_tokens, thinking, and so on, if any changed.

### 6.3 Tool-call pairing
Every `tool_use` block (in a reply) is indexed by `id`. When a later request carries a `tool_result` with a matching `tool_use_id`, the two are linked. Each tool call then knows: its name and input, its result (or "pending"), whether the result was an error, which exchange asked for it and which one delivered the result, and the time between them.

### 6.4 Statistics
- **Per exchange:** input, output, cache-read, and cache-write tokens (from `usage`); latency (time to first byte and total duration); cost; stop_reason.
- **Totals for the session:** requests, tokens by type, cost, total model time, errors.
- **Tool usage:** calls per tool, error rate per tool, average result size.
- **Cost:** computed from a **configurable price table** per model (input, output, cache read, cache write). If a model is unknown, show "–" rather than guess.

## 7. UI

### 7.1 Layout
```
┌────────────────────────────────────────────────────────────────────────────┐
│ ● live │ session ▾ │ 142 req · 3.1M in / 48k out · $4.12 · 11m │ ⚠ 3 errors│
├────────────────────────────────────────────────────────────────────────────┤
│ Filters: [tool ▾ Read, Bash] [kind ▾] [errors only ☐] [search…]            │
├──────────────────────────────────────────────────────┬─────────────────────┤
│ FEED (newest first)                                  │ STATS (collapsible) │
│                                                      │                     │
│ ▾ #142  12:04:31  sonnet  ◌ streaming…  2 new msgs   │ Tokens / request    │
│ ▸ #141  12:04:22  sonnet  200  3.2s  48k/312  $0.03  │ ▁▂▂▃▅▃▆▇            │
│        ↳ tool_use: Bash `pytest -q`  → ✗ error       │ Cost over time      │
│ ▸ #140  12:04:19  haiku   200  0.4s  side call       │ Tool usage          │
│ ▸ #139  12:04:10  sonnet  ⚠ 529 overloaded           │ Read   ████ 31      │
│ ...                                                  │ Bash   ██ 12  (2✗)  │
│                                                      │ Edit   █ 7          │
└──────────────────────────────────────────────────────┴─────────────────────┘
```

- **Header:** live status, session picker, running totals, and an error counter. Clicking the counter jumps to and filters errors.
- **Feed:** one card per exchange, **newest at the top**.
- **Stats panel:** collapsible side panel with the per-request token chart, cost over time, and the tool usage table. Clicking a bar or row filters the feed. On narrow screens it moves below the header as a collapsible strip.

### 7.2 Card: collapsed summary row
One line, scannable: index, time, model, kind tag, status badge (spinner while streaming), duration, tokens in/out, cost, and a short "what happened" (`2 new msgs`, `tool_use: Read utils.py`, `end_turn`). Errors get a red left border and an icon. Side calls are dimmed. Small "Δ" chips show changes since the previous request (`system Δ`, `tools +1`, `history rewritten`).

### 7.3 Card: expanded
The request and reply are two clearly separated blocks. The parts of the request are distinct, labeled sections, each with its own collapse state, color band, and size label:

```
▾ #141  12:04:22  sonnet  200  3.2s
  ┌ REQUEST ───────────────────────────────────────────────────────────────┐
  │ ▸ Changes since #139      system unchanged · tools unchanged · +2 msgs │
  │ ▸ System instructions     12,431 chars · 3 blocks · cached             │
  │ ▸ Tools (38)              Read, Write, Edit, Bash, Grep, …             │
  │ ▸ Conversation so far     46 messages                                  │
  │ ▾ New message                                                          │
  │     tool_result → Bash `pytest -q`   ✗ is_error   (paired with #140)   │
  │       ▸ 1 failed, 23 passed … (84 lines)                               │
  │ ▸ Parameters              max_tokens 32000 · thinking on · stream      │
  ├ REPLY ─────────────────────────────────────────────────────────────────┤
  │ ▸ thinking (1,204 chars)                                               │
  │   The test fails because `parse_date` returns None for …               │
  │   ┌ tool_use · Edit  utils.py ───────────────┬ result (in #142) ─────┐ │
  │   │ ▸ input JSON                             │ ✓ ok · 1 line         │ │
  │   └──────────────────────────────────────────┴───────────────────────┘ │
  │   stop_reason: tool_use · in 48,113 (cache read 47,900) · out 312      │
  ├ RAW ───────────────────────────────────────────────────────────────────┤
  │ ▸ request JSON  ▸ reply (assembled)  ▸ raw SSE  ▸ headers              │
  └────────────────────────────────────────────────────────────────────────┘
```

Section details:
- **Changes since previous:** the diff from §6.2. Text diffs for system, added and removed tools, changed parameters. Collapsed if nothing changed.
- **System instructions:** shown **in full** when expanded (no truncation), monospace-friendly, with a "copy" button. Multiple system blocks are shown separately with their `cache_control` markers.
- **Tools:** a list of names. Each expands to its description and a readable rendering of its input schema (field, type, required, description), with a raw JSON toggle.
- **Conversation so far:** collapsed by default. When expanded, it is the same message rendering as everywhere else, virtualized for long histories.
- **New message:** expanded by default, because this is what's actually new in this request.
- **Reply:** expanded by default. Text, thinking (collapsed), and tool calls.
- **Raw:** the exact request JSON, the assembled reply, raw SSE, and headers (redacted).

### 7.4 Tool calls beside their results
- Every `tool_use` in a reply is rendered as a **side-by-side pair**: input on the left, result on the right. Before the result arrives, the right side shows "pending…", and it fills in live when the next request brings the result.
- The pair shows the result status (✓ ok / ✗ error), its size, and a link to the request that carried it.
- In the next request's "New message" section, the `tool_result` shows a back-reference to the call it answers.
- On narrow screens the pair stacks vertically.

### 7.5 Formatting rules
- **Long text is collapsible:** anything over a threshold (for example 20 lines or 2,000 chars) shows a preview with "show all (N lines)". Expanding shows the full text, never truncated.
- **JSON is readable:** pretty-printed, syntax-colored, and collapsible per node (a tree view for large objects), with a copy button. JSON nested inside strings (common in tool inputs and results) is detected and offered pretty-printed too.
- **Text:** Markdown rendered, with highlighted code blocks and a raw toggle.
- **Roles and parts are visually distinct:** consistent color and label for system, tools, user, assistant, tool_use, tool_result, thinking, and errors.
- **Images/documents:** a thumbnail or placeholder with type and size. Base64 data is never dumped inline.
- Long lines wrap. Code blocks scroll horizontally. Nothing breaks the layout.

### 7.6 Filtering and search
- **By tool:** a multi-select of tool names. It shows exchanges whose reply calls the tool or whose new message carries its result.
- **By kind:** conversation / side call / token count / other.
- **Errors only.**
- **By model.**
- **Text search** across message content (system and tool definitions excluded by default, with a toggle).
- Active filters appear as removable chips. The header totals say "filtered: N of M".

### 7.7 Errors made easy to spot
- Error exchanges get a red border, an icon, and a status badge in the summary row.
- `tool_result` with `is_error` is marked red in both the pair and the summary row.
- Streams cut off mid-reply are marked "incomplete".
- The header error counter jumps to the errors. Pressing `e` goes to the next error.

### 7.8 Live behavior (newest first)
- New exchanges appear **at the top**. A streaming exchange updates its card in place (spinner, then status, tokens, and duration once done).
- **Don't move content under the reader:** if the user has scrolled down or has a card open, the view stays anchored on what they're reading, and a "↑ N new" pill appears at the top. Clicking it scrolls to the top. When the user is already at the top, new cards just appear.
- **Expand/collapse state survives updates and re-renders.**
- New cards get a brief highlight.
- Connection status shows live, reconnecting, or disconnected. The page reconnects automatically and resyncs from a snapshot.

### 7.9 Scale and keyboard
- Must stay smooth with more than 500 exchanges. Use a virtualized feed, and load bodies only when a card opens.
- Keys: `j`/`k` next/previous card, `Enter` expand/collapse, `e` next error, `/` search, `t` filter by tool, `g` go to newest, `Esc` close or clear.

## 8. Security and privacy
- Everything binds to `127.0.0.1` only.
- Secrets are redacted in the proxy before events are emitted. The browser never receives full tokens.
- Log files contain full prompts and code. Keep them out of version control (`.gitignore`).

## 9. Acceptance criteria (v1)
1. With the proxy (plus the tap) running and the harness pointed at it, each new exchange appears **at the top** of the feed within about 1 s of being sent.
2. Expanding any request shows four clearly separated parts: **system instructions (in full), tools, conversation so far, new message**, plus parameters and raw views.
3. Expanding any reply shows the model's text and every tool call it requested.
4. Each tool call is shown beside its result. The result fills in live when the following request carries it, and errors are marked.
5. "Changes since previous" correctly reports system, tool, parameter, and message changes, including "history rewritten".
6. Statistics show tokens, cost, time per request, and tool usage, and clicking them filters the feed. Every message stays reachable and expandable.
7. Filtering by tool, by kind, and to errors only works and combines.
8. Errors (HTTP, stream, tool `is_error`, incomplete) are visible from the collapsed feed without opening anything.
9. Long text and large JSON are collapsed by default, readable when expanded, and never truncated.
10. The reader's scroll position is not disturbed by new arrivals. The "↑ N new" pill works.
11. Refreshing the page restores the full session.
12. Killing the dashboard, or a crash in the tap, has no effect on the harness's traffic.
13. No full API key or bearer token reaches the browser or the log.
14. Unknown blocks or events render as raw JSON instead of breaking the page.

## 10. Suggested milestones
1. **Tap:** add structured event emission to the proxy, failure-safe and redacted.
2. **Parser:** reassemble exchanges (SSE and JSON) and classify them. Capture a real Claude Code session as a test fixture.
3. **Derived data:** request part split, diff since the previous request, tool pairing, statistics.
4. **Static viewer:** feed and expanded cards against the fixture, with formatting rules applied.
5. **Live:** push channel, newest-first insertion, scroll anchoring, reconnect and snapshot.
6. **Filters, stats panel, error navigation.**
7. **Polish:** keyboard shortcuts, virtualization, narrow-screen layout.

## 11. Open decisions

| # | Question | Options | Recommendation |
|---|---|---|---|
| 1 | Tech stack | Python vs Node/TS (ideally matching the proxy's language if the tap lives in-process). Frontend: vanilla JS (no build step) vs React/Svelte | Open. The leaning is Python backend plus vanilla JS |
| 2 | Transport proxy → dashboard | JSONL file / in-process SSE / local socket | JSONL file |
| 3 | Streaming display | Show reply text token by token while it streams, or only once complete | The parser handles both. Try live partial text with real traffic, then decide |
| 4 | Price table source | Hard-coded defaults vs a user-edited config file | A config file with defaults, editable without code changes |
| 5 | Multiple concurrent sessions | One feed with session tags vs a session picker | Session picker, plus an "all sessions" option |
| 6 | Export/persistence | Export session as JSON/HTML; keep history across restarts | Persistence comes free with JSONL. Export later |
| 7 | Other providers (OpenAI/Codex) | Out of scope / adapter layer | Keep parsing behind a provider-adapter interface |
