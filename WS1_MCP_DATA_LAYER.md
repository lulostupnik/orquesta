# Workstation 1 — MCP Integration & Data Layer

**Owner:** Developer 1
**Focus:** All MCP server connections, raw data ingestion, and the foundational data models that every other workstation depends on.

---

## 1. Mission

This workstation is the **single source of truth**. Nothing in Orquesta works without real calendar data flowing in. Your job is to wire up the MCP servers, normalize the raw data into clean schemas, and expose it so Workstations 2–5 never touch an MCP directly.

---

## 2. MCP Servers to Integrate

### 2.1 Google Calendar MCP (Primary — Required for MVP)

**Purpose:** Pulls all calendar events, attendees, recurrence rules, and response statuses for every team member.

**Connection:**

```javascript
mcp_servers: [
  {
    type: "url",
    url: "https://calendarmcp.googleapis.com/mcp/v1", // or actual endpoint
    name: "google-calendar-mcp"
  }
]
```

**Data to extract per team member:**

- All events in a configurable date range (default: last 90 days for review periods)
- Event title, start/end, duration, attendees list, response status per attendee
- Recurrence rules (RRULE) and whether the event is part of a series
- Description/agenda field (used to detect agenda-less meetings)
- Organizer field (who owns each meeting)

**Polling strategy (MVP):**

- On app load: full pull for the configured review period (last quarter)
- On demand: when the chat or review engine requests fresh data
- Cache in SQLite with a `last_synced` timestamp per member

### 2.2 Google Drive MCP (Secondary — Enrichment)

**Purpose:** Pulls shared documents referenced in meeting invites to assess whether meetings have preparation artifacts (agendas, decks, docs). A meeting that links to a shared doc is more likely to be productive.

**Connection:**

```javascript
mcp_servers: [
  {
    type: "url",
    url: "https://drivemcp.googleapis.com/mcp/v1",
    name: "google-drive-mcp"
  }
]
```

**Data to extract:**

- For each meeting that has a link in its description: fetch doc metadata (title, last modified, type)
- Flag meetings as `has_preparation_doc: true/false`
- Do NOT pull doc contents — only metadata for performance signal

### 2.3 Future MCPs (v2 — Do NOT build now, but design schema to accommodate)

| MCP | What it adds to performance reviews |
|---|---|
| GitHub MCP | Commit frequency, PR velocity, review turnaround — correlate with meeting load |
| Slack MCP | Async communication patterns — who messages after hours, thread participation |
| Jira/Linear MCP | Ticket throughput, sprint velocity — tie meeting reduction to output gain |
| Gmail MCP | Email volume patterns — another proxy for overload |

**Your schemas must include nullable fields for these future data sources** so Workstation 3 (Review Engine) can incorporate them without schema migrations.

---

## 3. Data Schemas

### 3.1 `team_member`

```json
{
  "id": "string (uuid)",
  "name": "string",
  "email": "string",
  "role": "junior_eng | mid_eng | senior_eng | manager | director",
  "hourly_rate": "number — derived from role or custom override",
  "timezone": "string — IANA format, e.g. America/Sao_Paulo",
  "calendar_id": "string — Google Calendar ID",
  "focus_blocks": [
    { "day": "monday", "start": "09:00", "end": "12:00" }
  ],
  "manager_id": "string | null",
  "review_cycle": "quarterly | biannual | annual",
  "last_review_date": "date | null",
  "next_review_date": "date | null",
  "github_handle": "string | null — v2",
  "slack_id": "string | null — v2",
  "jira_id": "string | null — v2"
}
```

### 3.2 `calendar_event` (normalized from MCP)

```json
{
  "id": "string",
  "source": "google_calendar",
  "title": "string",
  "start": "ISO datetime",
  "end": "ISO datetime",
  "duration_minutes": "number",
  "attendees": [
    {
      "member_id": "string",
      "response_status": "accepted | declined | tentative | needsAction"
    }
  ],
  "organizer_id": "string",
  "is_recurring": "boolean",
  "recurrence_id": "string | null — groups recurring instances",
  "recurrence_rule": "string | null — RRULE",
  "description": "string | null",
  "has_agenda": "boolean — heuristic: description.length > 50 chars",
  "has_preparation_doc": "boolean — from Drive MCP enrichment",
  "linked_doc_ids": ["string"],
  "category": "one_on_one | team_sync | cross_functional | external | focus_block | other",
  "raw_data": "object — preserve the full MCP response for debugging"
}
```

### 3.3 `sync_log`

```json
{
  "id": "string",
  "member_id": "string",
  "mcp_source": "google_calendar | google_drive",
  "synced_at": "ISO datetime",
  "events_pulled": "number",
  "date_range_start": "date",
  "date_range_end": "date",
  "status": "success | partial | failed",
  "error": "string | null"
}
```

---

## 4. Event Categorization Logic

You must auto-categorize every event. Workstation 3 depends heavily on accurate categories for review generation.

```
Rules (applied in order):
1. If attendees.length == 2 AND one is the member's manager → "one_on_one"
2. If title contains "standup", "sync", "weekly", "team" → "team_sync"
3. If attendees include people outside the team → "cross_functional"
4. If attendees include external domains → "external"
5. If title contains "focus", "block", "no meetings" → "focus_block"
6. Else → "other"
```

Expose a `/api/events/recategorize` endpoint so Workstation 5 (Chat) can let the manager correct miscategorizations.

---

## 5. API Endpoints You Own

| Endpoint | Method | Returns | Consumers |
|---|---|---|---|
| `POST /api/sync/{member_id}` | POST | Triggers MCP pull for one member, returns sync status | WS5 (Chat) |
| `POST /api/sync/team` | POST | Triggers full team sync | WS4 (Frontend on load) |
| `GET /api/events?member_id=&start=&end=` | GET | Filtered event list, normalized | WS2 (Analytics), WS3 (Review) |
| `GET /api/events/recurring?team_id=` | GET | All recurring meeting series with aggregated stats | WS2 (Cost Engine) |
| `GET /api/team/members` | GET | Team roster with metadata | All workstations |
| `PATCH /api/events/{id}/category` | PATCH | Recategorize an event | WS5 (Chat) |
| `GET /api/sync/status` | GET | Last sync times and health per member | WS4 (Frontend status indicator) |

---

## 6. Demo Mode

For the demo, you must support two modes controlled by an environment variable:

- **`MODE=live`** — Actually calls MCP servers. Used when connected to real Google accounts.
- **`MODE=demo`** — Loads from pre-built JSON files. No MCP calls. Zero failure risk.

In demo mode, load 5 team members with 90 days of calendar history from JSON fixtures. The fixture data must include the problematic patterns described in the main PRD (Ana overloaded, Carlos neglected, etc.).

**Both modes must serve identical API responses.** Downstream workstations must not know or care which mode is active.

---

## 7. Handoff Contract

You are DONE when:

- [ ] `POST /api/sync/team` pulls and normalizes data from Google Calendar MCP (or loads fixtures in demo mode)
- [ ] `GET /api/events` returns clean, categorized events for any member and date range
- [ ] `GET /api/events/recurring` returns grouped recurring meeting data with per-instance attendance rates
- [ ] Google Drive MCP enrichment adds `has_preparation_doc` to events
- [ ] SQLite stores all synced data with proper indices on `member_id`, `start`, `category`
- [ ] Sync log tracks every pull attempt
- [ ] Demo fixture JSON files cover all 5 team members, 90 days, all required problematic/healthy patterns
- [ ] All endpoints respond in <200ms from SQLite cache

---

## 8. What You Do NOT Build

- Cost calculations (Workstation 2)
- Analytics aggregation (Workstation 2)
- Review generation (Workstation 3)
- Any UI (Workstation 4)
- Chat interface or Claude prompts (Workstation 5)

You fetch, normalize, store, and serve raw data. That's it. Do it perfectly.
