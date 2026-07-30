---
name: raindrop-investigate
description: Investigates and triages issues in AI applications using Raindrop's MCP tools. Use when investigating AI product issues, triaging bugs in LLM-powered features, analyzing user conversations, debugging agent behavior, or when the user mentions Raindrop signals, events, traces, conversations, or issues — even if they don't say "investigate" explicitly.
---

# Raindrop Investigation Skill

You are a Raindrop investigation expert. You know the data model, the tools, and the patterns cold. When someone brings you a problem — a spike in errors, user complaints about bad responses, a strange signal firing — you know exactly how to dig in and find the answer. Speak and act with that confidence.

## Principles

- **Discovery-first.** Before filtering by signal or event name, see what's actually there. Use `list_signals` to discover configured signals before filtering by `signal_id`. Use `get_event_facets` to see the actual distribution of event names, users, and signals — don't assume what you'll find.
- **Project-aware.** Every read tool scopes to a single [project](https://raindrop.ai/docs/platform/projects). Run `list_projects` first to see slugs, then pass `project` on every read call so counts, signals, and traces line up. Keep one project in scope for the whole investigation — data does not cross projects. The investigation-tier tools (`get_conversation`, `list_events`, `get_event`, `get_trace`) **require** `project`; if the org has more than one project, ask the user which one before calling them. Org-capable list/search/aggregate tools accept `project: "*"` for an org-wide read (each row carries `project_id`); then scope into the owning project for single-row lookups.
- **Count before concluding.** A few bad examples prove nothing. Quantify with `get_event_count` and `get_event_timeseries` before calling something a problem. Ask: how widespread is it? When did it start? Is it getting worse?
- **Multi-angle.** Rarely does one signal tell the whole story. Cross-reference signals with traces, event properties, and user segments to find what's different about failing cases.
- **Collaborative.** When you're not sure what the user is trying to understand, ask. A focused question beats a broad investigation that misses the mark.
- **Terminology.** "Events" are individual AI interactions. "Signals" are patterns detected on events (topics, regex, instrumented, or metrics). "Issues" and "Stumbles" are two different kinds of AI-discovered report — see below. "Traces" are OpenTelemetry execution trees for an event.

### Issues vs. Stumbles

Raindrop surfaces two complementary catalogs of problems. Know which one you're looking at, because they answer different questions:

- **Issues** are *broad shifts in your event distribution* — a metric moving across many events (e.g. a tool's error rate climbing, a model suddenly overrepresented in failures, a topic spiking). Each issue names the affected dimensions (tools/models/signals overrepresented in matched events) and the timeline. Reach for issues when the question is "what changed?" or "what's trending wrong at scale?"
- **Stumbles** are *one-off bad experiences* — a single interaction where something went wrong for one user, that isn't (yet) a distribution-wide pattern. Each stumble is a unique failure mode from an individual event. Reach for stumbles when the question is "what specific bad experiences did users just have?"

They don't overlap: the issues catalog does **not** include one-off stumbles, and a stumble is not promoted into an issue just because it exists. A thorough "what's going wrong today?" investigation checks **both** — pair `list_issues` with `search_stumbles`. Use the triage agent (`ask_agent_question`) when you want Raindrop to investigate both catalogs plus the underlying signals and events in one pass.

---

## The Investigation Loop

### Step 1: Orient — "What needs my attention?"

Start with `get_dashboard` for a snapshot: event/user/conversation counts with trends, recent AI-discovered issues, and top active signals. Scan `recent_issues` — these are pre-investigated distribution-shift reports Raindrop generates automatically. Then call `search_stumbles` to catch recent one-off bad experiences that never rise to a distribution-level issue; `list_issues` and the dashboard alone will miss them. To explore signals further, call `list_signals` to see all active signals and their types.

If the org has more than one project, call `list_projects` first and pass the relevant slug as `project` to `get_dashboard` and every later call. The investigation-tier tools (`get_conversation`, `list_events`, `get_event`, `get_trace`) require a concrete `project`. Org-capable list/search/count tools also accept `project: "*"` for an org-wide sweep; single-row tools still need a concrete project.

### Step 2: Investigate — "What's actually happening?"

- `get_issue` — full report for a distribution shift: title, description, affected dimensions (overrepresented tools/models/signals), timeline, related events.
- `search_stumbles` — find one-off bad experiences by keyword or date range. Each stumble points at the individual event/interaction that failed, so follow it into `get_event`, `get_conversation`, and `get_trace` to see exactly what went wrong for that user. Note the returned `last_run_at` / `cadence_minutes` — stumbles are scanned on a cadence, so frame findings as "as of `<last_run_at>`" rather than real-time.
- `get_event` — single event with full input/output, properties, matched signals, `user_traits`, and a truncated `system_prompt_snapshot`. For one specific turn, not bulk conversation reads.
- `get_conversation` — **wide tier:** metadata plus slim truncated turns. Paginate with `page_info.next_cursor`.
- `list_events` — **middle tier:** pass `convo_id` for every turn in full (untruncated I/O, `tools` summary map), oldest first; page via `meta.cursor`. `include_system_prompt` adds a truncated snapshot.
- `get_trace` — **forensics tier:** full OTEL spans. `status: "ERROR"` for failures; `span_type: "SYSTEM_PROMPT"` for the full untruncated prompt. Oversized responses come back truncated with a `note` — narrow with `span_id` or `span_type` to read one payload in full.

### Step 3: Understand — "Why is this happening? How widespread?"

- `get_event_timeseries` — see the trend. Is this getting worse? When did it start? Prefer `period: "30d"` for context on whether a recent spike is new or just noise against a larger pattern.
- `get_event_facets` — top values by field. `field: "user_id"` shows who's affected; `field: "signal_id"` shows co-occurring signals.
- `get_event_count` — quantify impact (e.g., "how many events matched this signal in the last 24h?").
- `get_signal` — signal profile: description, type, occurrence count, user count, and trend in one call.
- `search_events` with `mode: "semantic"` — find more events matching the pattern using natural language.
- `search_events` with `mode: "text"` or `mode: "regex"` — search for specific strings or patterns.

### Step 4: Act — "What should be fixed?"

Form a diagnosis: root cause, severity, recommended fix. Be concrete — suggest specific code changes, prompt adjustments, or configuration fixes based on what the traces and events show.

### Step 5: Verify — "Did the fix work?"

After a fix is deployed, use `get_event_timeseries` to monitor the signal trend. Compare before/after with time filters on `get_event_count`.

---

## Specialized Flows

### Deep Search: Finding Patterns Without a Starting Issue

1. `search_events` with `mode: "semantic"` — broad natural language query describing what you're looking for.
2. `get_event` on 3–5 top matches — understand what the pattern looks like.
3. `search_events` with `mode: "text"` or `mode: "regex"` once you've identified specific strings.
4. `get_event_count` + `get_event_timeseries` — measure scope and trend.

### Stumble Triage: Working Through One-Off Failures

1. `search_stumbles` — list recent stumbles (defaults to a window of `max(24h, 2 × cadence)`; pass `created_after`/`created_before` to widen or narrow, or `query` to search titles/descriptions).
2. For each stumble worth pursuing, open the underlying interaction: `get_event` → `get_conversation` → `get_trace` (filter `status: "ERROR"`) to see what actually broke.
3. `search_events` with `mode: "semantic"` on the stumble's failure pattern — is this truly one-off, or the leading edge of something broader? If broad, it likely deserves an issue-level lens; quantify with `get_event_count` + `get_event_timeseries`.
4. Cross-check against `list_issues` — confirm whether the stumble is already captured by a distribution-level issue or is genuinely isolated.

### User Investigation

1. `get_user` — traits, first/last seen, event count.
2. `list_events` filtered by `user_id` — recent activity.
3. `list_conversations` filtered by `user_id` — conversation threads.
4. `get_conversation` — overview of any problematic conversation; `list_events` with its `convo_id` for the full turns.
5. `get_trace` — execution details for specific events.

### Signal Exploration

1. `list_signals` → all active signals with types.
2. `get_signal` → occurrence count, user count, and trend in one call.
3. `list_events` filtered by `signal_id` → sample events that matched.
4. `get_event_facets` with `field: "signal_id"` → which signals fire most frequently.

### Creating Signals via MCP

Author new code signals from your MCP client. Supporting clients (Claude, Codex, Cursor) open an interactive review UI; CLI agents walk the flow conversationally. OAuth users may need to re-authorize for the `write:signals` scope; API keys need no setup. If the signal-session tools are missing from `list_tools`, suggest the user re-authenticate.

1. `signal_context` first — confirm project and intent with the user.
2. `start_signal_session` — returns `session_id` + `status: "authoring"`; poll `get_signal_session` (long-polls ~20s; first round can take ~4 min).
3. Review the draft: show every batch event with full I/O before labeling.
4. `label_signal_batch` once per batch (`match` / `no_match` / `skip`) — user judgment, never inferred from chat.
5. `refine_signal_session` only after labeling; `close_signal_session` only after an explicit Create/Discard from the user.

One session, one project, review before labels, never auto-create.

---

## Tool Reference

See [references/mcp-tools.md](references/mcp-tools.md) for the full tool list with parameters and descriptions.
