# Workstation 3 — Performance Review Engine

**Owner:** Developer 3
**Focus:** The core product differentiator. You build the system that transforms MCP-sourced calendar analytics into structured, actionable performance reviews. This is why Orquesta exists.

---

## 1. Mission

Performance reviews today are based on vibes and recency bias. You are building the engine that replaces gut feeling with **data-backed performance narratives** generated from real calendar patterns pulled through MCP integrations.

A manager should be able to say: "Generate Ana's review" and get a comprehensive, honest, data-driven brief in under 10 seconds — covering meeting load, focus patterns, collaboration health, 1:1 consistency, and specific recommendations — all derived from the calendar data that MCP servers provide.

---

## 2. Dependencies

You consume data exclusively from Workstation 2's analytics API:

- `GET /api/analytics/review-data/{member_id}?review_period=quarter` — **your primary input** (the bundled payload with all metrics, trends, flags, comparisons, and history)
- `GET /api/analytics/person/{id}/history?weeks=12` — for detailed trend charts if needed

You also send prompts to the Claude API to generate narrative sections. You own all Claude prompt engineering for reviews.

---

## 3. Review Types

### 3.1 Pre-Review Brief (MVP — Primary deliverable)

Generated before a scheduled performance review. The manager reads this to prepare.

**Sections:**

1. **Header** — Name, role, review period, performance score badge (Healthy / Needs Attention / At Risk)
2. **Executive Summary** — 2-3 sentence overview: the single most important thing the manager should know
3. **Time Allocation Profile** — Meeting hours/week, focus ratio, how they compare to team
4. **Focus Time Analysis** — Trend over the quarter, direction, inflection points
5. **Meeting Quality Assessment** — Recurring meetings with declining attendance, agenda-less meetings, oversized meetings
6. **1:1 & Coaching Relationship** — Frequency, consistency, gaps, manager accountability note
7. **Risk Flags** — Active flags with data evidence
8. **Strengths Observed** — Calendar patterns that indicate healthy behaviors (protected focus blocks, reasonable decline rate, etc.)
9. **Recommendations for Discussion** — Specific, actionable items for the review conversation
10. **Cost Context** — What this person's meeting load costs in dollars, and what recovering time would save

### 3.2 Quick Performance Snapshot

A shorter format for ad-hoc checks between review cycles. Triggered by chat (Workstation 5) when the manager asks something like "How's Ana doing?"

**Sections:** Executive summary + key metrics + active flags + top recommendation. One screen, 30 seconds to read.

### 3.3 Team Review Summary

Aggregated view for when the manager is preparing for a full round of reviews.

**Sections:**

- Ranked list of team members by urgency (who needs attention first)
- Cross-team patterns (e.g., "3 of 5 engineers have declining focus time — this is systemic, not individual")
- Manager self-assessment (1:1 consistency, coaching gaps)
- Total optimization potential (hours and dollars recoverable)

### 3.4 Post-Review Action Tracker (v2 — schema only for now)

After a review, the manager commits to actions ("weekly 1:1s with Carlos", "remove Ana from Wednesday sync"). This tracker monitors follow-through.

```json
{
  "review_id": "string",
  "member_id": "string",
  "commitments": [
    {
      "id": "string",
      "type": "schedule_1on1 | remove_from_meeting | add_focus_block | reduce_meeting_load",
      "description": "Weekly 1:1 with Carlos every Tuesday",
      "target_metric": "one_on_one_gap_days < 10",
      "status": "active | completed | drifted | failed",
      "created_at": "date",
      "check_dates": ["date"],
      "compliance_rate": "number 0-1"
    }
  ]
}
```

Design the schema now. WS1 will check calendar data against commitments in v2.

---

## 4. Claude Prompt Engineering

This is the most critical part of your work. The quality of the review depends entirely on your prompts.

### 4.1 System Prompt — Review Generation

```
You are Orquesta's Performance Review Engine. You generate data-driven performance
briefs for engineering managers based on calendar analytics from MCP-connected data sources.

PRINCIPLES:
1. Every claim must cite specific numbers from the data provided. Never generalize
   without evidence. "Focus time is declining" must become "Focus time dropped from
   52% to 35% over Q1, a 17 percentage point decline."

2. Be honest but constructive. If someone is overloaded, say so clearly — but frame
   it as a systemic/structural issue first, not a personal failing.
   "Ana's output is constrained by meeting load, not skill" is better than
   "Ana attends too many meetings."

3. Always include dollar costs. "$2,400/month in low-attendance recurring meetings"
   is more compelling than "some meetings have low attendance."

4. Recommendations must be specific and actionable. Not "reduce meeting load" but
   "Exit the Wednesday Product Sync ($600/month, <50% attendance) and the Thursday
   QBR Prep (can be shortened from 2hrs to 1hr, saving $300/month)."

5. Include manager accountability. If 1:1 gaps exist, that's on the manager, not the
   report. Say so diplomatically: "1:1 cadence averaged 16 days this quarter — above
   the recommended 10-day interval. Consider protecting a recurring weekly slot."

6. Compare to team baselines. "27 hours/week of meetings" means nothing in isolation.
   "27 hours/week — 42% above the team average of 19" tells the story.

7. Flag strengths, not just problems. If someone protects their focus time well,
   say so. Reviews that only list problems are demoralizing and inaccurate.

DATA SOURCES:
The data you receive comes from MCP-connected integrations:
- Google Calendar MCP: meeting events, attendance, recurrence, time patterns
- Google Drive MCP: whether meetings have preparation documents
- Future (v2): GitHub MCP (commit patterns), Slack MCP (async communication), Jira MCP (ticket throughput)

Acknowledge the current data scope honestly: "This review is based on calendar and
meeting data. It captures time allocation and collaboration patterns but not code output,
ticket velocity, or qualitative work quality — those dimensions should be discussed
directly in the review."

OUTPUT FORMAT:
Return a JSON object with the following structure (each field is a string of markdown):
{
  "executive_summary": "2-3 sentences...",
  "time_allocation": "paragraph with metrics...",
  "focus_analysis": "paragraph with trend...",
  "meeting_quality": "paragraph with specific meetings...",
  "coaching_relationship": "paragraph on 1:1s...",
  "risk_flags": "bullet list of active flags with data...",
  "strengths": "bullet list of positive patterns...",
  "recommendations": "numbered list of specific actions...",
  "cost_context": "paragraph with dollar figures...",
  "data_scope_note": "brief note on what data sources were used..."
}
```

### 4.2 System Prompt — Quick Snapshot

Shorter version of the above. Instruct Claude to produce:

```json
{
  "status_badge": "healthy | needs_attention | at_risk",
  "one_liner": "single sentence summary",
  "key_metrics": { "focus_ratio, meeting_hours, days_since_1on1" },
  "top_flag": "most important flag or null",
  "top_recommendation": "single most impactful action"
}
```

### 4.3 System Prompt — Team Review Summary

```
Generate a team-wide review preparation brief. Analyze the team as a system, not
just a collection of individuals. Look for:
- Systemic patterns (is the whole team trending toward more meetings?)
- Distribution fairness (is meeting load concentrated on a few people?)
- Manager coverage (are all reports getting consistent 1:1s?)
- Organizational waste (recurring meetings that nobody finds valuable)

Rank team members by review urgency: who needs the most attention first?
End with the total optimization potential in hours and dollars.
```

### 4.4 Prompt Construction

When calling Claude, construct the prompt as:

```
[System prompt from above]

---

TEAM MEMBER DATA:
{JSON from /api/analytics/review-data/{member_id}}

---

Generate the performance review brief.
```

**Token budget:** Keep the review-data JSON payload under 4,000 tokens. The response should be under 2,000 tokens. Total round-trip should complete in <8 seconds.

---

## 5. Review Calibration Rules

Claude's output must be post-processed for consistency. Apply these rules after generation:

### Status Badge Assignment

| Condition | Badge |
|---|---|
| Performance score ≥ 70, no HIGH severity flags | **Healthy** |
| Performance score 40–69, OR any MEDIUM severity flag | **Needs Attention** |
| Performance score < 40, OR any HIGH severity flag | **At Risk** |

### Consistency Checks

- If `focus_ratio` is improving but the review says "declining" → flag for re-generation
- If `one_on_one_gap_days < 10` but the review flags 1:1 gaps → strip that section
- If no flags are active but badge is "At Risk" → demote to "Needs Attention"
- Dollar figures in the narrative must match the data payload (±5% for rounding)

### Sensitivity Rules

- Never use language like "underperforming", "failing", or "problematic" about a person
- Frame issues as structural: "constrained by", "impacted by", "could benefit from"
- Always include at least one strength, even for At Risk reviews
- 1:1 gap commentary must reflect on the manager, not the report

---

## 6. API Endpoints You Own

| Endpoint | Method | Returns | Consumers |
|---|---|---|---|
| `POST /api/review/generate/{member_id}` | POST | Full pre-review brief (calls Claude) | WS4 (Review panel), WS5 (Chat) |
| `POST /api/review/snapshot/{member_id}` | POST | Quick performance snapshot | WS5 (Chat — "how's Ana doing?") |
| `POST /api/review/team-summary` | POST | Team-wide review preparation brief | WS4 (Review panel), WS5 (Chat) |
| `GET /api/review/history/{member_id}` | GET | Past generated reviews (stored) | WS4 (Review history) |
| `POST /api/review/commit` | POST | Save post-review action commitments (v2 schema, store only) | WS5 (Chat) |
| `GET /api/review/commitments/{member_id}` | GET | Active commitments for a member | WS4 (Tracker view — v2) |

### Request Body for Generate

```json
{
  "review_period": "Q1 2026",
  "include_sections": ["all"],  // or subset: ["executive_summary", "recommendations"]
  "comparison_mode": "team_average",  // or "previous_quarter" or "both"
  "tone": "constructive",  // default, only option for now
  "format": "full"  // or "brief" for snapshot
}
```

### Response Body

```json
{
  "review_id": "uuid",
  "member_id": "string",
  "member_name": "string",
  "period": "Q1 2026",
  "generated_at": "ISO datetime",
  "status_badge": "needs_attention",
  "performance_score": 38,
  "sections": {
    "executive_summary": "markdown string...",
    "time_allocation": "markdown string...",
    "focus_analysis": "markdown string...",
    "meeting_quality": "markdown string...",
    "coaching_relationship": "markdown string...",
    "risk_flags": "markdown string...",
    "strengths": "markdown string...",
    "recommendations": "markdown string...",
    "cost_context": "markdown string...",
    "data_scope_note": "markdown string..."
  },
  "raw_metrics": { "...subset of review-data bundle for reference..." },
  "data_sources": ["google_calendar_mcp", "google_drive_mcp"],
  "model": "claude-sonnet-4-6",
  "generation_time_ms": 4200
}
```

---

## 7. Caching & Storage

- Store every generated review in SQLite with a `reviews` table
- Reviews are immutable once generated — a new generation creates a new record
- Cache the last review per member — if regenerated within 24 hours with no new sync data, return cached
- For the demo: pre-generate reviews for all 5 team members and cache them as fallbacks if Claude API is slow

---

## 8. Demo Script Support

Your engine must reliably produce the content shown in the PRD's demo Act 4:

> **Performance Context Brief — Ana Oliveira**
>
> Meeting load: 27 hrs/week average this quarter (team average: 19). 42% above team norm.
> Focus time: Dropped from 52% in January to 35% in April. Trend: declining.
> Recurring meeting debt: 3 recurring meetings with attendance below 60%. Combined cost: $1,800/month.
> 1:1 history: Average every 10 days (healthy range). Last one: 16 days ago (slightly overdue).
> Recommendation: Ana's output capacity is constrained by meeting load, not skill. Recovering 6 hours/week of focus time by dropping 2 low-value recurring meetings could significantly improve delivery.

If live Claude generation doesn't match this quality, the cached fallback must.

---

## 9. Handoff Contract

You are DONE when:

- [ ] `POST /api/review/generate/{id}` produces a full, multi-section review brief with accurate data citations
- [ ] `POST /api/review/snapshot/{id}` produces a quick snapshot in <3 seconds
- [ ] `POST /api/review/team-summary` produces a ranked team overview
- [ ] Status badges are correctly assigned based on score + flags
- [ ] Consistency checks catch and correct Claude hallucinations
- [ ] Reviews are stored and retrievable from history
- [ ] Cached fallback reviews exist for all 5 demo team members
- [ ] Dollar figures in narratives match the data ±5%
- [ ] Sensitivity rules are enforced (no negative person-language)
- [ ] Data scope note accurately reflects which MCP sources were used
- [ ] End-to-end generation completes in <10 seconds

---

## 10. What You Do NOT Build

- MCP connections or data fetching (Workstation 1)
- Metric calculations or cost math (Workstation 2)
- Any UI components (Workstation 4)
- Chat interface, scheduling logic, or recommendation ranking (Workstation 5)

You take numbers in and produce narratives out. The narratives must be honest, data-grounded, and useful enough that a manager can walk into a review meeting with only your brief.
