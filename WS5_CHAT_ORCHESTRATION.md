# Workstation 5 — Chat Interface, Recommendations & Orchestration

**Owner:** Developer 5
**Focus:** The conversational AI layer — the chat panel, proactive recommendations, scheduling/rescheduling logic, and the orchestration that ties the entire system together. You are the glue between the manager's intent and every other workstation.

---

## 1. Mission

You build the **brain and the mouth** of Orquesta. The manager talks to you in natural language. You understand what they want, pull data from Workstations 1-3 via their APIs, reason with Claude, and return structured, actionable responses. You also generate proactive recommendations without being asked.

The chat panel is the primary way managers interact with performance data. "Give me Ana's review" goes through you. "Schedule Carlos's 1:1" goes through you. "What should I change?" goes through you.

---

## 2. Dependencies

You are the only workstation that calls ALL other workstations:

| API | Source | Why you call it |
|---|---|---|
| `GET /api/events` | WS1 | Calendar data for scheduling logic |
| `POST /api/sync/{member_id}` | WS1 | Trigger fresh data pull when manager asks |
| `GET /api/analytics/person/{id}` | WS2 | Metrics for quick answers |
| `GET /api/analytics/flags/team` | WS2 | Proactive alerts |
| `GET /api/cost/recurring` | WS2 | Kill/shorten recommendations |
| `GET /api/cost/team/summary` | WS2 | Cost answers |
| `POST /api/review/generate/{id}` | WS3 | Full review generation |
| `POST /api/review/snapshot/{id}` | WS3 | Quick snapshots |
| `POST /api/review/team-summary` | WS3 | Team-wide review prep |

---

## 3. Chat Interface

### 3.1 UI Component (embedded in WS4's layout)

```
┌──────────────────────────────────┐
│  Orquesta Assistant               │
│  ● MCP Connected                  │
├──────────────────────────────────┤
│                                  │
│  [Recommendation cards]          │
│  ┌────────────────────────────┐  │
│  │ ⚠ Carlos hasn't had a 1:1 │  │
│  │ in 18 days. Schedule one?  │  │
│  │ [Find Slots] [Dismiss]     │  │
│  └────────────────────────────┘  │
│                                  │
│  ─── conversation ───            │
│                                  │
│  User: Give me a performance     │
│  brief for Ana before her review │
│                                  │
│  Assistant: Generating Ana's     │
│  review brief...                 │
│  [Review renders inline or       │
│   opens in review panel]         │
│                                  │
├──────────────────────────────────┤
│  [Type a message...]      [Send] │
└──────────────────────────────────┘
```

**Chat features:**

- Streaming responses (SSE from backend)
- Message history within session (not persisted across reloads for MVP)
- Action buttons embedded in responses (Schedule, Cancel, View Review, etc.)
- Recommendation cards pinned to top of chat, updated on dashboard load
- Loading state with stage indicators during review generation

### 3.2 Backend Chat Endpoint

`POST /api/chat`

```json
// Request
{
  "message": "Give me a performance brief for Ana before her review",
  "conversation_history": [
    { "role": "user", "content": "..." },
    { "role": "assistant", "content": "..." }
  ],
  "context": {
    "current_page": "/",
    "selected_member_id": null
  }
}

// Response (streamed via SSE)
{
  "type": "text" | "review" | "schedule_proposal" | "action_confirmation" | "recommendation",
  "content": "...",
  "actions": [
    { "label": "Schedule Option A", "endpoint": "/api/schedule", "payload": {...} },
    { "label": "View Full Review", "route": "/reviews/ana-oliveira" }
  ]
}
```

---

## 4. Intent Classification & Routing

When a message comes in, classify the intent and route to the right backend flow:

| Intent | Detection keywords | Route |
|---|---|---|
| **Generate review** | "review", "performance brief", "review prep", "prepare review" | Call WS3 `POST /api/review/generate/{id}` |
| **Quick status check** | "how's [name] doing", "status of [name]", "check on [name]" | Call WS3 `POST /api/review/snapshot/{id}` |
| **Team review prep** | "team reviews", "prepare all reviews", "review cycle", "who needs attention" | Call WS3 `POST /api/review/team-summary` |
| **Schedule** | "schedule", "book", "set up", "find time" | Scheduling engine (section 5) |
| **Reschedule** | "reschedule", "move", "shift", "adjust", "[name] has emergency/conflict" | Rescheduling engine (section 6) |
| **Analyze** | "analyze", "how's the team", "calendar health", "time breakdown" | Call WS2 analytics, format with Claude |
| **Optimize** | "optimize", "what should I change", "save money", "reduce meetings" | Optimization engine (section 7) |
| **Cost query** | "how much", "cost of", "what are we spending" | Call WS2 cost endpoints, format response |
| **Data refresh** | "sync", "refresh", "update data", "pull latest" | Call WS1 `POST /api/sync/team` |

### Intent Classification Prompt

```
Classify the user's message into exactly one intent:
- generate_review (asking for a performance review or brief for a specific person)
- quick_status (casual check on how someone is doing)
- team_review_prep (preparing for a round of reviews across the team)
- schedule (booking a new meeting or 1:1)
- reschedule (moving an existing meeting)
- analyze (understanding team patterns or metrics)
- optimize (asking for actionable changes to improve team performance)
- cost_query (asking about meeting costs)
- data_refresh (asking to update or sync calendar data)

Also extract: mentioned_names[], time_references[], constraints[]

Respond as JSON only.
```

---

## 5. Scheduling Engine

When the intent is `schedule`:

### 5.1 Constraint Resolution

1. Pull calendars for all involved people from WS1: `GET /api/events?member_id=X&start=&end=`
2. Identify free slots within the requested time range
3. Filter out slots that violate constraints:
   - Focus blocks (from team member config) — never schedule over these
   - Timezone overlap windows (compute intersection of working hours)
   - Back-to-back prevention (at least 15min gap before/after)
   - Explicit constraints from the user ("not Friday afternoon", "morning preferred")
4. Rank remaining slots by quality:
   - Preserves most focus time → higher rank
   - Within core working hours → higher rank
   - Doesn't create back-to-back for either party → higher rank

### 5.2 Response Format

```
I found 2 options for your 1:1 with Ana:

**Option A:** Tuesday 2pm (Buenos Aires) / 2pm (São Paulo)
- Cost: $80 (30 min × $75 + $85)
- ✅ No focus time disrupted
- ✅ No back-to-back for either of you

**Option B:** Wednesday 3pm (Buenos Aires) / 3pm (São Paulo)
- Cost: $80
- ⚠ Follows Ana's team sync — she'll have 3 hours of consecutive meetings

Recommendation: Option A is better for Ana's focus day.

[Schedule Option A]  [Schedule Option B]  [Show more options]
```

### 5.3 Event Creation

When the manager clicks "Schedule Option A":

1. Call `POST /api/schedule` (WS1 writes to local JSON/SQLite in demo mode)
2. Return confirmation with the event details
3. Trigger a data refresh for affected members
4. WS4 animates the new event into the calendar view

---

## 6. Rescheduling Engine

When the intent is `reschedule`:

1. Identify the meeting to move (fuzzy match on title + person + date)
2. Identify why it's being moved (constraint from user message)
3. Find new slots using the same scheduling engine
4. Check for cascading conflicts (does moving meeting A create a conflict for meeting B?)
5. Present diff: old time → new time, with cost impact
6. On confirmation: update event, trigger refresh

---

## 7. Optimization Engine (Performance-Focused)

When the manager asks "What should I change?" — this is where the product shines.

### 7.1 Data Collection

Pull from WS2:

- `GET /api/cost/recurring` — all recurring meetings with waste metrics
- `GET /api/analytics/flags/team` — all active flags
- `GET /api/analytics/team/comparison` — per-person metrics vs averages

### 7.2 Optimization Categories

**Meeting Kills** (descheduling):

```
Criteria for recommending a meeting kill:
- attendance_rate < 50% for 3+ weeks
- monthly_cost > $500
- OR: no agenda for 30+ days AND monthly_cost > $300
```

**Meeting Shortens:**

```
Criteria:
- Scheduled duration > actual typical duration by 30%+
  (detected if meetings consistently end early — v2 with meeting end time tracking)
- OR: >6 attendees AND duration > 60min AND category = "team_sync"
```

**1:1 Gaps:**

```
Criteria:
- one_on_one_gap_days > 14 for any report
- Prioritize by: days overdue × performance_score_inverse
  (longer gap + worse health = more urgent)
```

**Schedule Shifts:**

```
Criteria:
- Meetings scheduled during peak focus hours (first 2 hours of workday)
- Meetings outside timezone overlap windows
```

**Focus Block Creation:**

```
Criteria:
- Any person with focus_ratio < 40% who doesn't have explicit focus blocks
- Recommend 2-3 hour blocks on their lightest meeting days
```

### 7.3 Optimization Brief Format

```
## Performance Optimization Brief

Based on your team's calendar data from the last 30 days:

### Kill (3 meetings — save $4,200/month)
1. **Wednesday Product Sync** — $2,400/month, <50% attendance for 3 weeks.
   Replace with async Slack update. [Kill It]
2. **Friday Retrospective** — $1,200/month, no agenda in 45 days.
   Assess value or cancel. [Kill It]
3. **Ad-hoc Design Review** — $600/month, only 2 of 6 people regularly attend.
   Make it optional. [Kill It]

### Fix (2 actions — recover 8 hrs/week)
4. **Move daily standup** from 9am to 10:30am — protects morning focus for
   3 team members. Estimated gain: 5 hrs/week. [Reschedule]
5. **Schedule Carlos's 1:1** — 18 days overdue, PR velocity declining.
   3 slots available this week. [Find Slots]

### Protect (1 action)
6. **Create focus blocks for Ana** — Tue/Thu mornings, 3 hours each.
   Her focus ratio is 35% (team avg: 52%). [Create Blocks]

### Impact Summary
- Monthly savings: **$4,200**
- Weekly hours recovered: **31 hours** across the team
- 1:1 coverage: **75% → 100%**
- Projected team focus ratio: **48% → 58%**
```

---

## 8. Proactive Recommendations

Generated on dashboard load without the manager asking. These appear as cards pinned above the chat conversation.

### 8.1 Generation Flow

On every dashboard load (or every 30 minutes):

1. Call `GET /api/analytics/flags/team` — get active flags
2. Call `GET /api/cost/recurring` — get waste candidates
3. Rank by urgency: HIGH flags first, then cost savings, then improvements
4. Take top 3-5 recommendations
5. Format as cards with one-click actions

### 8.2 Recommendation Card Schema

```json
{
  "id": "rec_001",
  "type": "urgent_1on1 | kill_meeting | schedule_change | focus_protection | review_due",
  "severity": "high | medium | low",
  "title": "Carlos hasn't had a 1:1 in 18 days",
  "description": "PR velocity dropped this week. 3 available slots.",
  "impact": "$0 cost, high performance impact",
  "actions": [
    { "label": "Find Slots", "type": "schedule", "payload": {...} },
    { "label": "Dismiss", "type": "dismiss" }
  ]
}
```

### 8.3 Review-Specific Recommendations

When reviews are approaching (within 14 days of `next_review_date`):

```
🔵 Performance reviews in 8 days
Ana Oliveira — At Risk — Review brief ready [View Brief]
Carlos Mendez — Needs Attention — [Generate Brief]
Diego Ramirez — Healthy — [Generate Brief]
```

These take priority over all other recommendations.

---

## 9. Claude System Prompts

### 9.1 Main Chat System Prompt

```
You are the Orquesta performance assistant. You help engineering managers
optimize their team's time and prepare data-driven performance reviews
using calendar data from MCP-connected sources (Google Calendar, Google Drive).

CORE BEHAVIOR:
1. Always include dollar costs in any recommendation or analysis.
2. Always compare individual metrics to team averages.
3. When asked about a person, lead with performance-relevant insights.
4. When scheduling, always explain constraint reasoning.
5. When recommending changes, quantify the impact.

CONTEXT YOU HAVE:
- Team roster with roles, rates, and timezones
- Calendar events from Google Calendar MCP
- Meeting costs computed from salary bands
- Focus time ratios and trends
- 1:1 history and gaps
- Active flags per person

CONTEXT YOU DON'T HAVE (be honest about this):
- Code output, PR velocity, ticket throughput (coming in v2 via GitHub/Jira MCP)
- Qualitative work quality
- Personal circumstances
- Actual meeting content or outcomes

PERFORMANCE REVIEW FOCUS:
When the conversation touches on performance, reviews, or individual assessment:
- Frame everything as data-driven insight, not judgment
- Always include the data scope disclaimer
- Recommend specific calendar changes, not vague advice
- Calculate the dollar impact of every recommendation
- Note strengths alongside concerns
- Attribute 1:1 gaps to the manager, not the report

RESPONSE STYLE:
- Concise. No filler. Lead with the insight.
- Use markdown formatting for structure.
- Include action buttons where applicable.
- Show your reasoning for scheduling decisions.
```

### 9.2 Context Injection

Every chat message to Claude must include:

```
TEAM DATA (current as of {last_sync_time}):
{team_members JSON with roles and rates}

RELEVANT CALENDAR DATA:
{events for mentioned members, scoped to relevant time range}

ACTIVE FLAGS:
{current flags for mentioned members}

ANALYTICS SNAPSHOT:
{key metrics for mentioned members}
```

Keep context under 6,000 tokens. Summarize older conversation history aggressively.

---

## 10. Fallback & Demo Safety

### 10.1 Pre-cached Responses

For every demo act, store a pre-built response that matches the demo script exactly:

| Demo Act | User prompt | Cached response key |
|---|---|---|
| Act 2 | "Schedule 30-minute 1:1s with Ana and Diego..." | `demo_schedule_1on1s` |
| Act 3 | "Diego has a client emergency..." | `demo_reschedule_diego` |
| Act 4 | "Give me a calendar-based performance brief for Ana" | `demo_review_ana` |
| Act 5 | "What should I change about how my team spends time?" | `demo_optimization_brief` |

### 10.2 Fallback Logic

```
1. Send prompt to Claude API with 8-second timeout
2. If response arrives within timeout → use live response
3. If timeout OR error → serve cached response with no visual indicator
   (the audience should not know it's cached)
4. Log the fallback event for debugging
```

### 10.3 Response Quality Check

Before displaying a live Claude response, verify:

- Contains dollar figures (at least one)
- Doesn't contradict the data (quick check: if data says Ana has 27 hrs/week and response says 15, reject)
- Doesn't contain apologies or refusals
- Is within reasonable length (not too short, not >2000 tokens)

If any check fails → serve cached response.

---

## 11. API Endpoints You Own

| Endpoint | Method | Returns | Consumers |
|---|---|---|---|
| `POST /api/chat` | POST (SSE) | Streamed chat response | WS4 (Chat panel) |
| `GET /api/recommendations` | GET | Top 3-5 proactive recommendation cards | WS4 (Chat panel header) |
| `POST /api/schedule` | POST | Event creation confirmation | WS4 (Chat action buttons) |
| `POST /api/reschedule` | POST | Event move confirmation with diff | WS4 (Chat action buttons) |
| `POST /api/optimize` | POST | Full optimization brief | WS4 (Chat panel) |

---

## 12. Handoff Contract

You are DONE when:

- [ ] Chat panel sends messages, receives streamed responses, and renders action buttons
- [ ] Intent classification correctly routes to the right backend flow for all 9 intent types
- [ ] "Generate review" intent triggers WS3 and renders the review in chat or navigates to review page
- [ ] Scheduling engine resolves constraints, ranks options, and presents proposals with costs
- [ ] Rescheduling engine handles conflicts and shows before/after diffs
- [ ] Optimization brief covers kills, fixes, and protections with dollar impacts
- [ ] Proactive recommendations load on dashboard init and prioritize review-related alerts
- [ ] Review-due recommendations appear when reviews are approaching
- [ ] Cached fallback responses exist for all 5 demo acts
- [ ] Fallback logic fires transparently within 8 seconds
- [ ] Response quality check catches bad Claude outputs
- [ ] Full demo script (Acts 1-5) can be performed end-to-end through the chat

---

## 13. What You Do NOT Build

- MCP connections or data normalization (Workstation 1)
- Metric calculations, cost formulas, or flag logic (Workstation 2)
- Review prompt engineering or narrative generation (Workstation 3)
- Dashboard UI, charts, or review page layout (Workstation 4)

You orchestrate. You route. You converse. You recommend. You make the other four workstations sing together.
