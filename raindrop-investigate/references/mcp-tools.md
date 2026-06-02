# Raindrop MCP Tools Reference

## Contents

- [Events](#events)
- [Conversations](#conversations)
- [Users](#users)
- [Signals](#signals)
- [Traces](#traces)
- [Issues & Stumbles](#issues--stumbles)
- [Docs & Feedback](#docs--feedback)

Auth: org API key or OAuth 2.1 token (via PropelAuth introspection).

**Time ranges:** All tools use a `period` string parameter (e.g. `"1h"`, `"24h"`, `"7d"`, `"30d"`) rather than explicit start/end timestamps. Max lookback is 90 days.

**Pagination:** Most list tools use `cursor` (not `offset`) for pagination. `search_stumbles` uses a page number.

---

## Events

### `raindrop_list_events`
Paginated event list with optional filters. Sorted most-recent first.

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | int (1–100) | Max results (default: 25) |
| `cursor` | string | Pagination cursor from previous response |
| `user_id` | string | Filter by user |
| `convo_id` | string | Filter by conversation |
| `event_name` | string | Filter by event name |
| `signal_id` | string | Filter by signal ID |
| `period` | string | How far back to look (default: `"24h"`) |

### `raindrop_get_event`
Single event by ID. Returns full input, output, properties, and matched signals.

| Parameter | Type | Description |
|-----------|------|-------------|
| `event_id` | string | Required |

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

### `raindrop_get_event_count`
Aggregate event count with filters. Use for quantifying impact.

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_id` | string | Filter by user |
| `convo_id` | string | Filter by conversation |
| `event_name` | string | Filter by event name |
| `signal_id` | string | Filter by signal |
| `period` | string | How far back to count (default: `"24h"`) |

### `raindrop_get_event_timeseries`
Event counts bucketed by time interval. Use for trend analysis. Ensure `period` is wider than `interval` (e.g., use `"7d"` or wider for daily buckets).

| Parameter | Type | Description |
|-----------|------|-------------|
| `interval` | `"minute"` \| `"hour"` \| `"day"` \| `"week"` \| `"month"` | Bucket size (default: `"day"`) |
| `user_id` | string | Filter by user |
| `event_name` | string | Filter by event name |
| `signal_id` | string | Filter by signal |
| `period` | string | Time range (default: `"7d"`) |

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

---

## Conversations

### `raindrop_list_conversations`
Paginated conversation list. Sorted by most recent message first.

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | int (1–100) | Max results (default: 25) |
| `cursor` | string | Pagination cursor |
| `user_id` | string | Filter by user |

### `raindrop_get_conversation`
Single conversation with full message thread.

| Parameter | Type | Description |
|-----------|------|-------------|
| `conversation_id` | string | Required |
| `include_events` | boolean | Include conversation events (default: `true`) |
| `event_limit` | int (1–100) | Max events to include (default: 50) |

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

### `raindrop_get_user`
Single user with traits, first/last seen timestamps, and event count.

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_id` | string | Required |

---

## Signals

### `raindrop_get_dashboard`
Snapshot of your application: event/user/conversation counts with period-over-period trends, recent AI-discovered issues, and top active signals. Start every investigation here.

| Parameter | Type | Description |
|-----------|------|-------------|
| `period` | string | Time window (default: `"24h"`) |

### `raindrop_list_signals`
All signals for your organization (topics, regex, instrumented, metrics). Sorted by most recently created.

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | int (1–100) | Max results (default: 25) |
| `cursor` | string | Pagination cursor |

### `raindrop_get_signal`
Signal metadata + occurrence count + timeseries trend in a single call. Use to understand a signal's scope and trajectory before diving into events.

| Parameter | Type | Description |
|-----------|------|-------------|
| `signal_id` | string | Required |
| `period` | string | Time window for counts and timeseries (default: `"24h"`) |

Returns: signal definition, `occurrence_count`, and a `timeseries` array (hourly for periods ≤24h, daily otherwise).

### `raindrop_list_signal_groups`
Signal groups for your organization. Groups are collections of related signals (e.g. "errors", "performance").

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | int (1–100) | Max results (default: 25) |
| `cursor` | string | Pagination cursor |

### `raindrop_get_signal_group`
Single signal group with its member signals.

| Parameter | Type | Description |
|-----------|------|-------------|
| `group_id` | string | Required |

---

## Traces

### `raindrop_get_trace`
OpenTelemetry trace spans for an event or trace ID. Returns the full span tree: LLM calls, tool calls, and internal spans.

Provide `event_id` or `trace_id` — if both are provided, `event_id` takes precedence.

| Parameter | Type | Description |
|-----------|------|-------------|
| `event_id` | string | Event ID to get traces for |
| `trace_id` | string | OpenTelemetry trace ID to look up directly |
| `span_type` | `"INTERNAL"` \| `"LLM_GENERATION"` \| `"LLM_GENERATION_STREAM"` \| `"TOOL_CALL"` | Filter to a specific span type |
| `status` | `"UNSET"` \| `"OK"` \| `"ERROR"` | Filter by span status — use `"ERROR"` to find failures |
| `limit` | int (1–200) | Max spans to return (default: 50) |

---

## Issues & Stumbles

### `raindrop_list_issues`
Issue Detection reports for broader changes in the organization's event distribution. For broad questions such as "what issues happened today?", also call `raindrop_search_stumbles`.

> Note: Issue Detection requires the Pro plan.

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | int (1–100) | Max results (default: 25) |
| `cursor` | string | Pagination cursor |
| `period` | string | How far back to look (default: `"90d"`, max: `"90d"`) |

### `raindrop_get_issue`
Full AI-generated investigation report for an Issue Detection distribution change.

| Parameter | Type | Description |
|-----------|------|-------------|
| `issue_id` | string | Required |

### `raindrop_search_stumbles`
Search one-off unique failure modes from individual interactions. Stumbles are cases where something bad happened once, not broader changes in the event distribution. For broad questions such as "what issues happened today?", call this alongside `raindrop_list_issues`.

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | string | Case-insensitive text search across title, subtitle, and description |
| `created_before` | ISO timestamp | Find Stumbles created before this time |
| `created_after` | ISO timestamp | Find Stumbles created after this time. Defaults to `max(24h, 2 x cadence_minutes)` |
| `page` | int (≥1) | Page number (default: 1). Each page returns up to 50 Stumbles |

Returns cadence metadata so callers can describe the result as "as of `<last_run_at>`" instead of implying real-time detection.

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
