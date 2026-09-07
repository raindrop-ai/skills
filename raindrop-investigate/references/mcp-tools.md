# Raindrop MCP Tools Reference

## Contents

- [Projects](#projects)
- [Events](#events)
- [Conversations](#conversations)
- [Users](#users)
- [Signals](#signals)
- [Signal authoring (MCP_SIGNAL)](#signal-authoring-mcp_signal)
- [Traces](#traces)
- [Issues](#issues)
- [Stumbles](#stumbles)
- [Docs & Feedback](#docs--feedback)

Auth: org API key or OAuth 2.1 token (via PropelAuth introspection).

**Time ranges:** All tools use a `period` string parameter (e.g. `"1h"`, `"24h"`, `"7d"`, `"30d"`) rather than explicit start/end timestamps. Max lookback is 90 days.

**Pagination:** All list tools use `cursor` (not `offset`) for pagination. The cursor is returned in each response.

**Projects:** Most read tools accept an optional `project` parameter that scopes the call to a single [project](https://raindrop.ai/docs/platform/projects). Omitting `project` (or passing `"default"`) reads from the org's built-in **Production** project on aggregate/list tools. **Required on investigation-tier tools:** `raindrop_get_conversation`, `raindrop_list_events`, `raindrop_get_event`, and `raindrop_get_trace` — call `raindrop_list_projects` first; if the org has more than one project, ask the user which slug to use. Multi-project orgs pass a project slug to target one project at a time; reads are isolated per project. An unknown or archived slug is rejected.

**All-projects reads:** Pass `*` as `project` on the org-capable read tools — `raindrop_list_events`, `raindrop_search_events`, `raindrop_get_event_count`, `raindrop_get_event_timeseries`, `raindrop_get_event_facets`, `raindrop_list_conversations`, and `raindrop_list_users` — to read across all active projects; rows carry a `project_id`. Single-row lookups (`raindrop_get_event`, `raindrop_get_conversation`, `raindrop_get_trace`, signals, issues, stumbles, dashboard) still require a concrete project (not `*`).

---

## Projects

### `raindrop_list_projects`
List the projects in your organization. Pass a returned `project` value to any read tool's `project` parameter to scope the call to that project. The `is_default` field marks the project used when `project` is omitted. Single-project orgs only ever see the default **Production** project here.

| Parameter | Type | Description |
|-----------|------|-------------|
| (none) | | Returns `project` (slug), `name`, and `is_default` for each active project |

---

## Events

### `raindrop_list_events`
Paginated event list with optional filters. Each row is a full event: untruncated input/output, model, signals, feature flags, properties, and a `tools` map (`{ count, total_duration_ms?, error_count? }`).

Without `convo_id`: newest first, default `period: "30d"`. With `convo_id`: oldest first, no time window — page via `meta.cursor` until `has_more` is false. Use this after `raindrop_get_conversation` instead of calling `raindrop_get_event` per turn.

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | int (1–100) | Max results (default: 25) |
| `cursor` | string | Pagination cursor from previous response |
| `user_id` | string | Filter by user |
| `convo_id` | string | Conversation ID — full untruncated turns, oldest first; page via `meta.cursor` |
| `event_name` | string | Filter by event name |
| `signal_id` | string | Filter by signal ID |
| `model` | string | Filter by AI model name |
| `feature_flags` | array | Filter by feature flag key/value pairs |
| `properties` | array | Filter by event properties, e.g. `[{ key: 'status', op: 'eq', value: 'error' }]` |
| `user_traits` | array | Filter by user trait key/value pairs |
| `include_system_prompt` | boolean | Truncated prompt snapshot (default: `false`). With `convo_id`: one top-level snapshot + `system_prompt_snapshot_turn_id`; without: per-row snapshots |
| `period` | string | How far back to look (default: `"30d"`). **Ignored when `convo_id` is set.** |
| `project` | string | **Required.** Project slug from `raindrop_list_projects`, or `*` for org-wide read |

### `raindrop_get_event`
Single event by ID. Returns full input, output, properties, and matched signals, plus per-event enrichment: `user_traits` and a truncated `system_prompt_snapshot`. Use for one specific turn; to read all turns of a conversation use `raindrop_list_events` with `convo_id`.

| Parameter | Type | Description |
|-----------|------|-------------|
| `event_id` | string | Required |
| `project` | string | **Required.** Project slug from `raindrop_list_projects` |

### `raindrop_search_events`
Search events by text, regex, or semantic similarity. Use `mode: "semantic"` to find events matching a natural language description — this is the primary tool for pattern discovery.

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | string | Search query |
| `mode` | `"text"` \| `"semantic"` \| `"regex"` | Search mode (default: `"text"`) |
| `limit` | int (1–100) | Max results (default: 25) |
| `cursor` | string | Pagination cursor |
| `user_id` | string | Filter by user |
| `event_name` | string | Filter by event name |
| `period` | string | How far back to search (default: `"24h"`) |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project, or pass `*` to read across all the org's active projects |

### `raindrop_get_event_count`
Aggregate event count with filters. Use for quantifying impact.

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_id` | string | Filter by user |
| `convo_id` | string | Filter by conversation |
| `event_name` | string | Filter by event name |
| `signal_id` | string | Filter by signal |
| `period` | string | How far back to count (default: `"24h"`) |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project, or pass `*` to read across all the org's active projects |

### `raindrop_get_event_timeseries`
Event counts bucketed by time interval. Use for trend analysis. Ensure `period` is wider than `interval` (e.g., use `"7d"` or wider for daily buckets).

| Parameter | Type | Description |
|-----------|------|-------------|
| `interval` | `"minute"` \| `"hour"` \| `"day"` \| `"week"` \| `"month"` | Bucket size (default: `"day"`) |
| `user_id` | string | Filter by user |
| `event_name` | string | Filter by event name |
| `signal_id` | string | Filter by signal |
| `period` | string | Time range (default: `"7d"`) |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project, or pass `*` to read across all the org's active projects |

### `raindrop_get_event_facets`
Top values for a field across events with counts. Use to understand distribution before filtering.

| Parameter | Type | Description |
|-----------|------|-------------|
| `field` | `"event_name"` \| `"user_id"` \| `"signal_id"` | Field to facet |
| `limit` | int (1–100) | Top N values (default: 20) |
| `user_id` | string | Filter by user |
| `event_name` | string | Filter by event name |
| `signal_id` | string | Filter by signal |
| `period` | string | How far back to look (default: `"24h"`) |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project, or pass `*` to read across all the org's active projects |

---

## Conversations

**Three-tier read model:** `raindrop_get_conversation` (wide, truncated turns) → `raindrop_list_events` with `convo_id` (middle, full I/O) → `raindrop_get_event` / `raindrop_get_trace` (single-turn enrichment / forensics). All three require `project`.

### `raindrop_list_conversations`
Paginated conversation list. Sorted by most recent message first.

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | int (1–100) | Max results (default: 25) |
| `cursor` | string | Pagination cursor |
| `user_id` | string | Filter by user |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project, or pass `*` to read across all the org's active projects |

### `raindrop_get_conversation`
Conversation overview: metadata plus slim chronological turns (event `id`, `event_name`, `timestamp`, truncated I/O, tool-call counts). Start here, then go deeper with `raindrop_list_events` (`convo_id`) for full turns or `raindrop_get_trace` on a turn's `id`. No system prompt here — use `raindrop_get_trace` with `span_type: "SYSTEM_PROMPT"`.

| Parameter | Type | Description |
|-----------|------|-------------|
| `conversation_id` | string | Required |
| `event_limit` | int (1–100) | Max turns to include (default: 50) |
| `cursor` | string | Pagination cursor from a previous response's `page_info.next_cursor` |
| `project` | string | **Required.** Project slug from `raindrop_list_projects` |

Returns `page_info` (`total_messages`, `returned`, `has_more`, `next_cursor`) for paging through long conversations.

---

## Users

### `raindrop_list_users`
Paginated user list.

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | int (1–100) | Max results (default: 25) |
| `cursor` | string | Pagination cursor |
| `user_id` | string | Filter to a specific user ID |
| `order_by` | `"last_seen"` \| `"first_seen"` | Sort field (default: `"last_seen"`) |
| `order_direction` | `"asc"` \| `"desc"` | Sort direction (default: `"desc"`) |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project, or pass `*` to read across all the org's active projects |

### `raindrop_get_user`
Single user with traits, first/last seen timestamps, and event count.

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_id` | string | Required |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project |

---

## Signals

### `raindrop_get_dashboard`
Snapshot of your application: event/user/conversation counts with period-over-period trends, recent AI-discovered issues, and top active signals. Start every investigation here.

| Parameter | Type | Description |
|-----------|------|-------------|
| `period` | string | Time window (default: `"24h"`) |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project |

### `raindrop_list_signals`
All signals for your organization (topics, regex, instrumented, metrics). Sorted by most recently created.

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | int (1–100) | Max results (default: 25) |
| `cursor` | string | Pagination cursor |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project |

### `raindrop_get_signal`
Signal metadata + occurrence count + timeseries trend in a single call. Use to understand a signal's scope and trajectory before diving into events.

| Parameter | Type | Description |
|-----------|------|-------------|
| `signal_id` | string | Required |
| `period` | string | Time window for counts and timeseries (default: `"24h"`) |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project |

Returns: signal definition, `occurrence_count`, and a `timeseries` array (hourly for periods ≤24h, daily otherwise).

### `raindrop_list_signal_groups`
Signal groups for your organization. Groups are collections of related signals (e.g. "errors", "performance").

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | int (1–100) | Max results (default: 25) |
| `cursor` | string | Pagination cursor |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project |

### `raindrop_get_signal_group`
Single signal group with its member signals.

| Parameter | Type | Description |
|-----------|------|-------------|
| `group_id` | string | Required |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project |

### Signal authoring (MCP_SIGNAL)

Create new code signals from MCP (requires the `MCP_SIGNAL` feature flag). OAuth users re-authorize for the `write:signals` scope; API keys need no setup. If these tools are missing from `list_tools`, suggest the user re-authenticate.

| Tool | Purpose |
|------|---------|
| `raindrop_signal_context` | **Call first.** Loads the authoring workflow. |
| `raindrop_start_signal_session` | Start authoring. Returns `session_id` + `status: "authoring"`; reuse the same `project` + `session_id` throughout. |
| `raindrop_get_signal_session` | Poll while authoring (long-polls ~20s) until `reviewing`, `ready`, or `failed`. |
| `raindrop_get_signal_session_status` | Lightweight status check. |
| `raindrop_get_signal_session_code` | Classifier source — only when the user explicitly asks. |
| `raindrop_label_signal_batch` | Label every batch event once (`match` / `no_match` / `skip`). Required before refine or close. |
| `raindrop_refine_signal_session` | Tighten boundaries after labeling; returns to authoring. |
| `raindrop_close_signal_session` | `outcome: "create"` + `confirm: true`, or `outcome: "discard"`. Never auto-create. |

### Existing signal refinement

| Tool | Purpose |
|------|---------|
| `raindrop_refine_signal` | Refine an existing accepted user JavaScript signal with event labels or a comment, automatically applying the resulting definition to that same signal. Returns immediately with `status: "refining"`. |

`raindrop_refine_signal` requires `signal_id` (UUID) and `project`. `org` is
optional. `positive_event_ids` and `negative_event_ids` default to empty arrays
and accept at most 500 event IDs each. `comment` defaults to an empty string
and accepts at most 4,000 characters. Provide at least one event ID or a
nonblank comment; event IDs must belong to the selected project.

A successful call returns `signal_id` and `status: "refining"`. Call once,
acknowledge that refinement has started, and continue with the user's other
work. Raindrop completes and applies the refinement automatically in the
background.

---

## Traces

### `raindrop_get_trace`
OpenTelemetry trace spans for an event or trace ID. Returns the full span tree: LLM calls, tool calls, and internal spans. **Required:** `project`.

Provide `event_id` or `trace_id` — if both are provided, `event_id` takes precedence.

Pass `span_type: "SYSTEM_PROMPT"` to get the full untruncated system prompt for an event as a single synthetic span.

Oversized responses come back with span payloads truncated inline and a `note` — call again with `span_id` (or `span_type` / `status`) to read one payload in full.

| Parameter | Type | Description |
|-----------|------|-------------|
| `event_id` | string | Event ID to get traces for |
| `trace_id` | string | OpenTelemetry trace ID to look up directly |
| `span_id` | string | Return only this span — use after a truncated response |
| `span_type` | `"INTERNAL"` \| `"LLM_GENERATION"` \| `"LLM_GENERATION_STREAM"` \| `"TOOL_CALL"` \| `"SYSTEM_PROMPT"` | Filter to a specific span type; `"SYSTEM_PROMPT"` returns the full system prompt |
| `status` | `"UNSET"` \| `"OK"` \| `"ERROR"` | Filter by span status — use `"ERROR"` to find failures |
| `limit` | int (1–200) | Max spans to return (default: 50) |
| `project` | string | **Required.** Project slug from `raindrop_list_projects` |

---

## Issues

### `raindrop_list_issues`
AI-discovered investigation reports for **broad shifts in your event distribution** (a tool's error rate climbing, a model overrepresented in failures, a topic spiking). Automatically generated when Raindrop detects a distribution change worth investigating. This catalog does **not** include one-off [Stumbles](#stumbles) — pair it with `raindrop_search_stumbles` for a complete "what's going wrong?" picture.

> Note: requires the `new_issues_nav` feature flag to be enabled for your org.

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | `"active"` \| `"all"` | `"active"` excludes ignored/rejected issues (default: `"active"`) |
| `limit` | int (1–100) | Max results (default: 25) |
| `cursor` | string | Pagination cursor |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project |

### `raindrop_get_issue`
Full AI-generated investigation report. Includes title, description, tags, timeline of events, and related events.

| Parameter | Type | Description |
|-----------|------|-------------|
| `issue_id` | string | Required |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project |

---

## Stumbles

Stumbles are **one-off bad experiences** — a single interaction where something went wrong for one user, not a distribution-wide change. This is the counterpart to [Issues](#issues): issues answer "what changed at scale?", stumbles answer "what specific bad experiences did users just have?". The two catalogs don't overlap, so a thorough investigation queries both.

### `raindrop_search_stumbles`
Search stumbles by keyword or date range. Returns up to 50 per page, sorted most-recent first. Each stumble references the underlying event/interaction, so follow it into `raindrop_get_event`, `raindrop_get_conversation`, and `raindrop_get_trace` to see what actually broke.

The response also includes `cadence_minutes` and `last_run_at`: stumbles are detected on a scan cadence, not in real time, so frame findings as "as of `<last_run_at>`".

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | string | Text to match in stumble title, subtitle, and description (case-insensitive) |
| `created_after` | string (ISO) | Only stumbles created after this time. Defaults to `max(24h ago, 2 × cadence_minutes ago)` so low-cadence orgs cover at least two scan windows |
| `created_before` | string (ISO) | Only stumbles created before this time |
| `page` | int | Page number (default: 1); each page returns up to 50 |
| `project` | string | Scope to a project (from `raindrop_list_projects`); omit for the default project |

---

## Docs & Feedback

### `raindrop_search_docs`
Search Raindrop documentation. Use when you need to look up SDK usage, configuration options, or integration details.

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | string | Search query |

### `raindrop_submit_feedback`
Submit feedback to the Raindrop team. Posts directly to their internal channel.

| Parameter | Type | Description |
|-----------|------|-------------|
| `feedback` | string | Description of the issue, what didn't work, or what was unclear |
| `category` | `"bug"` \| `"docs"` \| `"unclear"` \| `"feature_request"` \| `"other"` | Feedback category |
