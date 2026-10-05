# Raindrop MCP Tools Reference

## Contents

- [Projects](#projects)
- [RQL](#rql)
- [Events](#events)
- [Costs](#costs)
- [Conversations](#conversations)
- [Users](#users)
- [Dashboards](#dashboards)
- [Signals](#signals)
- [Signal authoring (MCP_SIGNAL)](#signal-authoring-mcp_signal)
- [Existing signal refinement](#existing-signal-refinement)
- [Traces](#traces)
- [Agent simulation replays](#agent-simulation-replays)
- [Issues](#issues)
- [Stumbles](#stumbles)
- [Docs & Feedback](#docs--feedback)

Auth: org API key or OAuth 2.1 token (via PropelAuth introspection).

**Time ranges:** Most time-scoped read tools use a `period` string parameter (e.g. `"1h"`, `"24h"`, `"7d"`, `"30d"`). `raindrop_run_rql` accepts `time_range: {from, to}` for event and trace queries, with a maximum seven-day window and a default of the latest seven days. SQL predicates can narrow this window but cannot widen or move it. Keep raw-text searches within 24 hours with a selective predicate. User and conversation rollups represent lifetime or whole-conversation totals.

`query_cost` requires explicit `from` and `to` ISO timestamps, with a maximum window of 31 days (7 days for hourly trends).

Dashboard previews require an explicit `time_range: {from, to}` with ISO timestamps spanning at most seven days. Saved dashboards use their own time settings.

**Pagination:** List tools use `cursor` (not `offset`) for pagination. `raindrop_run_rql` uses `LIMIT` and has no cursor.

**Projects:** Most read tools accept an optional `project` parameter that scopes the call to a single [project](https://raindrop.ai/docs/platform/projects). Omitting `project` (or passing `"default"`) reads from the org's built-in **Production** project on aggregate/list tools. **Required on investigation-tier tools:** `raindrop_get_conversation`, `raindrop_list_events`, `raindrop_get_event`, and `raindrop_get_trace` — use a project slug provided by the user or already resolved in the current organization. Use `raindrop_list_projects` to discover projects or verify the selection when needed; ask the user if several projects could apply. Multi-project orgs pass a project slug to target one project at a time; reads are isolated per project. An unknown or archived slug is rejected.

**All-projects reads:** Pass `*` as `project` on the org-capable read tools — `raindrop_list_events`, `raindrop_search_events`, `raindrop_get_event_count`, `raindrop_get_event_timeseries`, `raindrop_get_event_facets`, `raindrop_list_conversations`, and `raindrop_list_users` — to read across all active projects; rows carry a `project_id`. Single-row lookups (`raindrop_get_event`, `raindrop_get_conversation`, `raindrop_get_trace`, signals, issues, stumbles, dashboard) still require a concrete project (not `*`).

---

## RQL

### `raindrop_skills`
Load the guide for the task: `rql` for counts, breakdowns, trends, and comparisons; `rql_reference` for the full RQL schema and functions; `explore` for individual records and semantic search; `dashboards` for creating and editing saved dashboards; `signals` for signal authoring; `evals` for eval workflows; and `triage` for explicit delegation to Raindrop Triage. Analytical questions can start with `rql` directly.

| Parameter | Type | Description |
|-----------|------|-------------|
| `topic` | string | Optional guide name; omit to list available guides |

### `raindrop_run_rql`
Run one read-only RQL `SELECT` over `events`, `traces`, `users`, or `conversations` in one concrete project. Use a project slug provided by the user or already resolved in the current organization. Use `raindrop_list_projects` to discover projects or verify the selection when needed. Use RQL for counts, breakdowns, trends, and comparisons. Load `raindrop_skills` with `topic: "rql"` directly for common event fields and checked examples; loading `explore` first is unnecessary. Use `topic: "rql_reference"` for the full table and function reference and additional examples. Keep semantic search, efficient dedicated rollups, signal tools, and event or trace detail tools for questions they answer better. Give computed expressions aliases that do not reuse source column names.

| Parameter | Type | Description |
|-----------|------|-------------|
| `org` | string | Optional organization selection |
| `project` | string | **Required.** One concrete project slug; `*` is not supported |
| `query` | string | Required RQL `SELECT`, 1–20,000 characters |
| `time_range` | object | Optional `{from, to}` ISO timestamps (inclusive start, exclusive end), at most seven days; defaults to the latest seven days for events and traces |

Count events with `uniqExact(event_id)` and spans with `uniqExact(tuple(trace_id, span_id))`. MCP enforces a maximum seven-day window for events and traces. Pass `time_range: {from, to}` for historical or narrower windows; omission selects the latest seven days. Choose 24 hours when the user gives no timeframe. SQL predicates can narrow that window but cannot widen or move it. Query successive windows for longer investigations and reuse the same explicit window while paging. The window does not apply to lifetime user or conversation rollups. Keep raw-text and serialized-payload searches within 24 hours with a selective predicate. MCP uses the same RQL compiler as Triage. A `LIMIT` caps returned rows, not scan cost. The default is 100 rows; the explicit maximum is 1,000.

Results include `columns`, `data`, `rowCount`, limit and truncation flags, `statistics`, the enforced `timeRange` for events and traces, default-window information, and supported `reference_targets`. `rowCount` counts returned rows before clipping, not all matching events. Report the time range and any clipping. Errors carry a message and source span for query correction; narrow a timed-out query before retrying. Content is redacted. Organizations with zero data retention enabled or unverified cannot run this tool.

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

## Costs

### `raindrop_query_cost` (`query_cost` on the server)

Analyze project-scoped LLM spend and token usage, using provider/gateway-reported costs plus catalog-priced usage.

| Operation | Result |
|-----------|--------|
| `summary` | Total spend, tokens, and pricing coverage for the window |
| `breakdown` | Cost by `model` or `provider`, ranked by total cost descending |
| `timeseries` | Chronological `hour` or `day` trend |
| `events` | Recent associated events with spend, newest first; paginate with `next_cursor` until null |

| Parameter | Type | Description |
|-----------|------|-------------|
| `operation` | `"summary"` \| `"breakdown"` \| `"timeseries"` \| `"events"` | Required |
| `from` | ISO datetime | Required; inclusive start |
| `to` | ISO datetime | Required; exclusive end, after `from`, at most 31 days later |
| `filters` | object | Optional exact `model` and/or `provider` string arrays (max 50 values per array, 500 characters per value); unsupported for timeseries |
| `group_by` | `"model"` \| `"provider"` | Required for breakdown |
| `interval` | `"hour"` \| `"day"` | Required for timeseries; hourly windows are limited to 7 days |
| `limit` | int (1–100) | Default 20; max 50 for breakdown and 100 for events |
| `cursor` | string | Event pagination cursor from `next_cursor`; keep the project, window, and filters unchanged |
| `org` | string | Optional organization selector; must be authorized |
| `project` | string | Project slug; omit for the default project. `*` is unsupported |

Results include `cost_basis: "reported_plus_catalog"`, a cost note, and `quality` with pricing coverage and caveats. Check `unpriced_model_calls` and `pricing_coverage_ratio` before quoting a total: missing usage or catalog coverage makes `total_cost_usd` incomplete, and an entirely unpriced total is null. Cache usage ratios are only meaningful when cache reporting is complete. Event results omit calls not associated with an event; summary quality reports those calls.

Cost attribution by user, conversation, or function is unsupported. Use `get_event` or `get_trace` after selecting a cost event to investigate it. Events are ordered by recency, not spend.

Example:

```json
{
  "operation": "breakdown",
  "group_by": "model",
  "from": "2026-07-01T00:00:00.000Z",
  "to": "2026-07-08T00:00:00.000Z",
  "project": "default"
}
```

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

## Dashboards

Load `raindrop_skills` with `topic: "dashboards"` before authoring. Use `topic: "rql"` or `"rql_reference"` for query syntax. All tools accept optional `org` to select an accessible organization. Direct writes require an OAuth user and `write:dashboards`; API keys can read organization-shared dashboards but cannot create or edit them. Both write tools save immediately. Refresh cached tool inventories after the overview rename.

### `raindrop_list_dashboards`

List accessible dashboards, including the built-in Usage board. Omit `project` to list across projects, or pass a project to narrow the catalog. Pass `dashboard_id` or an exact `name` for a snapshot; `dashboard_id` wins. Catalog entries include IDs, titles, scope, links, and panel summaries, without query text. Named snapshots apply the saved `filters` and return them with the board. Each query result reports `skippedDashboardFilters` for filter kinds its source cannot apply. Private dashboards appear only to their owner.

| Parameter | Type | Description |
| --- | --- | --- |
| `org` | string | Optional organization selection. |
| `project` | string | Optional project selection. Omit to list across projects. |
| `dashboard_id` | string | Optional saved dashboard UUID or `default-usage`. |
| `name` | string | Optional case-insensitive exact title. |
| `period` | string | Optional snapshot range: `1h`, `6h`, `24h`, `3D`, `7D`, or `30D`. Defaults to the saved range. |

### `raindrop_get_dashboard`

Read a saved dashboard before editing. Returns `{data: ...}` containing `dashboard_id`, `project_id`, title, description, visibility, revision, `time_settings`, saved `filters`, panels as `{id, panel}` in layout order, and a scoped URL. The built-in Usage board is available through `list_dashboards` and cannot be edited.

| Parameter | Type | Description |
| --- | --- | --- |
| `org` | string | Optional organization selection. |
| `project` | string | Required concrete project; `*` is unsupported. |
| `dashboard_id` | string | Saved dashboard UUID. Supply this or `name`; ID wins when both are set. |
| `name` | string | Case-insensitive exact title. Ambiguous titles require selection by ID. |

### `raindrop_preview_dashboard_panel_query`

Validate and execute one exact panel query with dashboard time-range semantics. Preview does not apply dashboard-wide filters. Omit timestamp and end_timestamp filters from `WHERE` and `HAVING`; the dashboard supplies them. Match visualization fields to query output aliases. Count events with `uniqExact(event_id)` and spans with `uniqExact(tuple(trace_id, span_id))`.

| Parameter | Type | Description |
| --- | --- | --- |
| `org` | string | Optional organization selection. |
| `project` | string | Required concrete project owning the dashboard. |
| `query` | string | Required final RQL query, 1–20,000 characters. |
| `time_range` | object | Required `{from, to}` ISO timestamps, positive duration of at most seven days. |

Raw-text filters require at most 24 hours and a selective predicate. For longer saved ranges, preview a bounded subset and disclose it. Results include bounded rows, columns, query statistics, and preview metadata. They validate the query and do not establish the full dashboard's values or cost. Retention restrictions, redaction, query capacity controls, timeouts, and result limits apply. Narrow or simplify a timed-out query before retrying.

### `raindrop_create_dashboard`

Save a new organization-shared dashboard. The server assigns panel IDs and layout. Creation is not idempotent: after an uncertain response, check the catalog before retrying.

| Parameter | Type | Description |
| --- | --- | --- |
| `org` | string | Optional organization selection. |
| `project` | string | Required concrete owning project. |
| `title` | string | Required dashboard title. |
| `description` | string or null | Optional description. |
| `time_settings` | object | Optional dashboard range and refresh settings; defaults to the last seven days with refresh off. |
| `panels` | array | Required complete panels in display order. Query panels include `kind: "query"`, title, query, and visualization; text panels include `kind: "text"`, title, and body. |

Preview each exact query before saving. The result contains `success`, `ui_type: "dashboard_created"`, `dashboard_id`, `project_id`, title, revision, `panel_ids`, and URL.

### `raindrop_edit_dashboard`

Apply actions to an existing dashboard and save them atomically. Read `get_dashboard` first. Query updates include the complete query and visualization together; preview every new or changed query.

| Parameter | Type | Description |
| --- | --- | --- |
| `org` | string | Optional organization selection. |
| `project` | string | Required concrete owning project. |
| `dashboard_id` | string | Required saved dashboard UUID. |
| `expected_revision` | integer | Required revision from the latest definition read. |
| `summary` | string | Required short summary of the change. |
| `actions` | array | Required ordered edit actions. |

Supported actions are `add_panel`, `update_query_panel`, `update_text_panel`, `duplicate_panel`, `remove_panel`, `set_panel_size`, `tidy_layout`, `update_dashboard_details`, `set_filters`, and `set_time_settings`. Target existing panels by their returned `panel_id`. For size changes use `compact`, `standard`, `wide`, or `full`; never supply grid coordinates.

Use `set_filters` for whole-dashboard defaults. Its `filters` array replaces the complete saved list, so preserve filters the user did not ask to remove; `[]` clears all filters. For example, `{"action":"set_filters","filters":[{"kind":"userId","values":["example-user"]}]}` scopes supported panels to that user without changing their queries. Defaults persist for everyone opening the dashboard. Combine filters and panels in the same edit, or use only `set_filters` for a filter-only request; no query preview is needed when queries stay unchanged. For a new dashboard, create it first, then set filters using the returned revision. Filter kinds include properties, user traits, feature flags, signals, event names, conversation IDs, user IDs, models, tool names/counts, error counts, and error status. Discover uncertain keys and values before saving.

Line and bar visualizations accept `seriesColors`, a map of legend labels to six-digit hex values. Only set it when the user names colors; otherwise use `palette`. Labels use readable field names without a group, the group alone with one numeric field, and `<group> · <field>` with several. Overlay groups start with the query name or ref and append the series-field value when present. Matching ignores case and treats underscores as spaces.

On a revision conflict, reload and rebuild the actions. Do not just change `expected_revision`. Validation failures save nothing. The result contains `success`, `ui_type: "dashboard_edit_proposal"`, `dashboard_id`, `project_id`, summary, revision, `affected_panel_ids`, and URL. When actions include `set_filters`, the result also returns the saved `filters`, including `[]` when cleared. Otherwise it omits that field. Despite that response label, the edit is already saved.

---

## Signals

### `raindrop_get_application_overview`
Snapshot of your application: event/user/conversation counts with period-over-period trends, recent AI-discovered issues, and top active signals. Use this for an application overview. This tool was renamed from `raindrop_get_dashboard`; that name now reads a saved dashboard definition.

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
| `raindrop_skills` | **Call first** with `topic: "signals"`. Loads the signal workflow. |
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
| `raindrop_refine_signal` | Refine an existing accepted user JavaScript signal, usually using false positives found during a signal investigation. Also accepts direct feedback through event labels or a comment. Automatically applies the resulting definition to the same signal; returns `status: "refining"` when started. |

`raindrop_refine_signal` requires `signal_id` (UUID) and `project`. `org` is
optional. `positive_event_ids` and `negative_event_ids` default to empty arrays
and accept at most 500 event IDs each. `comment` defaults to an empty string
and accepts at most 4,000 characters. Provide at least one event ID or a
nonblank comment; event IDs must belong to the selected project. OAuth callers
need `write:signals`; API-key callers can also use this write tool.

Use `negative_event_ids` for the false positives already identified. Add a
`comment` when it helps explain the desired change. Positive examples are optional; do
not gather them just to satisfy a refinement step. Reuse known scope and
events, and accept direct refinement requests without requiring discovery.

A successful call returns `signal_id` and `status: "refining"`. Call once,
acknowledge that refinement has started, and continue with the user's other
work. Raindrop completes and applies the refinement automatically in the
background.

This uses the same headless refinement flow as the Signals page. No polling or
`close_signal_session` call is needed.

---

## Traces

### `raindrop_get_trace`
OpenTelemetry trace spans for an event or trace ID. Returns the full span tree: LLM calls, tool calls, and internal spans. **Required:** `project`.

Provide `event_id` or `trace_id` — if both are provided, `event_id` takes precedence.

Pass `span_type: "SYSTEM_PROMPT"` to get the full untruncated system prompt for an event as a single synthetic span.

Oversized responses come back with span payloads truncated inline and a `note` — call again with `span_id` (or `span_type` / `status`) to read one payload in full.

For a suspicious tool call, re-fetch its `span_id` with `include_context: true` to compare the
call's input/output with the model decision before it and the answer afterward. This adds up
to eight ancestors and the nearest preceding/following model generations in the closest
parent branch with a neighbor. Context comes from the anchor's trace in the same project,
including other events in that trace. It is returned alongside the anchor in chronological
`data`, with IDs identifying the relationships in `context`. These are temporal neighbors,
not proof of causation. Existing `span_type`, `status`, and `limit` apply to the anchor lookup;
extra context is not filtered by them.

```json
{"project":"support","event_id":"<event-id>","span_id":"<refund-span-id>","include_context":true}
```

Discovery is bounded to 24 hours before/after the anchor's start and 2,000 spans. Check
`context.incomplete`, `scan_limit_reached`, `ancestor_limit_reached`, and the returned
`window_start_ns` / `window_end_ns`. Missing parent records or capped reads can leave context
incomplete. A null neighbor means none was found within those bounds, not that no model call
occurred. If the scan cap is exceeded, only the anchor is returned; use ordinary `get_trace`
with `cursor` to inspect the trace. To re-read a truncated payload, use its `span_id` without
`include_context`.

| Parameter | Type | Description |
|-----------|------|-------------|
| `event_id` | string | Event ID to get traces for |
| `trace_id` | string | OpenTelemetry trace ID to look up directly |
| `span_id` | string | Return this span; optionally add related spans with `include_context` |
| `include_context` | boolean | With `span_id`, include ancestors and nearby model generations in the same branch. Default false. Cannot combine with `cursor` or `SYSTEM_PROMPT`. |
| `span_type` | `"INTERNAL"` \| `"LLM_GENERATION"` \| `"LLM_GENERATION_STREAM"` \| `"TOOL_CALL"` \| `"SYSTEM_PROMPT"` | Filter to a specific span type; `"SYSTEM_PROMPT"` returns the full system prompt |
| `status` | `"UNSET"` \| `"OK"` \| `"ERROR"` | Filter by span status — use `"ERROR"` to find failures |
| `limit` | int (1–200) | Max spans to return (default: 50) |
| `project` | string | **Required.** Project slug from `raindrop_list_projects` |

---

## Agent simulation replays

### `raindrop_get_replay_trace`
Recorded spans for ONE event replay (from `replay_event`, `run_replay_suite`, or `get_simulation_review` candidate replay IDs): the replayed agent's model generations, prompts, and tool calls. `get_trace` does not see replays. Narrow with `span_type` (`LLM_GENERATION`, `TOOL_CALL`) or `status` `ERROR`; page with `meta.cursor` until `meta.has_more` is false, keeping the same filters. If a payload is truncated, re-read that `span_id`. `data.spans` is null while the replay is still running or when no recording was kept; call `get_replay_progress` until it finishes.

| Parameter | Type | Description |
|-----------|------|-------------|
| `replay_id` | UUID string | Required replay ID |
| `span_type` | `"INTERNAL"` \| `"LLM_GENERATION"` \| `"LLM_GENERATION_STREAM"` \| `"TOOL_CALL"` | Filter to a span type |
| `status` | `"UNSET"` \| `"OK"` \| `"ERROR"` | Filter by span status |
| `span_id` | string | Filter to one span |
| `limit` | int (1–200) | Max spans to return (default: 50) |
| `cursor` | string | Pagination cursor; reuse the same filters on each page |
| `org` | string | Organization reference from `raindrop_list_organizations`; omit for the default organization |

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
