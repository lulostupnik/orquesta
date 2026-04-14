# Workstation 2 — Analytics Engine & Cost Engine

**Owner:** Developer 2
**Focus:** All computed metrics, cost calculations, performance scoring, trend analysis, and flag generation. You turn raw calendar events (from Workstation 1) into the numbers that power dashboards (Workstation 4) and reviews (Workstation 3).

---

## 1. Mission

You are the **math brain** of Orquesta. Every dollar figure, every focus ratio, every burnout flag, every trend line originates from your code. Workstation 3 cannot generate a single performance review without your analytics. Workstation 4 cannot render a single chart without your aggregations. Get the numbers right.

---

## 2. Dependencies

You consume data exclusively from Workstation 1's API:

- `GET /api/events?member_id=&start=&end=` — individual event data
- `GET /api/events/recurring?team_id=` — recurring meeting series
- `GET /api/team/members` — team roster with roles and hourly rates

You do NOT call any MCP servers directly. If you need data, it comes from WS1's endpoints.

---

## 3. Cost Engine

### 3.1 Meeting Cost Calculation

```
meeting_cost = SUM(attendee_hourly_rate for each attending attendee) × (duration_minutes / 60)
```

**Rules:**

- Only count attendees with `response_status` of `accepted` or `needsAction` (assume they attended)
- `declined` attendees are excluded from cost
- `tentative` attendees are counted at 50% rate (they show up roughly half the time)
- If an attendee is not in the team roster (external), use a configurable default rate ($50/hr)

### 3.2 Recurring Meeting Costs

```
monthly_cost = meeting_cost × occurrences_per_month
annual_cost  = monthly_cost × 12
```

For recurring meetings, also compute:

- `effective_monthly_cost` — factors in actual attendance rate: `monthly_cost × avg_attendance_rate`
- `waste_cost` — the delta: `monthly_cost - effective_monthly_cost` (cost of empty chairs)

### 3.3 Salary Bands (configurable defaults)

| Role | Default Hourly Rate |
|---|---|
| Junior Engineer | $30/hr |
| Mid Engineer | $50/hr |
| Senior Engineer | $75/hr |
| Manager | $85/hr |
| Director | $120/hr |

Store in a config that the manager can override per person.

---

## 4. Person Analytics

Computed per team member, per period (week, month, quarter). **Quarter is critical for performance reviews.**

### 4.1 Core Metrics

| Metric | Calculation | Used by |
|---|---|---|
| `meeting_hours` | Sum of attended meeting durations | Dashboard, Review |
| `focus_hours` | `total_work_hours - meeting_hours` (assume 8hr workday, 5 days) | Dashboard, Review |
| `focus_ratio` | `focus_hours / total_work_hours` | Dashboard, Review, Flags |
| `meeting_cost` | Sum of this person's share of all meeting costs | Dashboard, Review |
| `one_on_one_last_date` | Most recent 1:1 with their manager | Dashboard, Flags |
| `one_on_one_gap_days` | Days since last 1:1 | Dashboard, Flags, Review |
| `one_on_one_count` | Number of 1:1s in the period | Review |
| `one_on_one_avg_interval` | Average days between 1:1s | Review |
| `back_to_back_streaks` | Count of days with 3+ consecutive meetings (no gap > 15min) | Flags |
| `meetings_organized` | Meetings where this person is the organizer | Review |
| `meetings_attended` | Total meetings attended | Review |
| `decline_rate` | Meetings declined / total invited | Review |
| `agenda_less_meetings` | Recurring meetings attended that have no agenda | Review |

### 4.2 Trend Metrics (for performance reviews)

These compare the current period against the previous period. **Workstation 3 depends on these heavily.**

| Metric | Calculation | Review impact |
|---|---|---|
| `focus_ratio_trend` | Current quarter focus_ratio vs. previous quarter | "Declining focus time" is a key review signal |
| `focus_ratio_delta` | Percentage point change | "+5pp" or "-12pp" |
| `meeting_hours_trend` | `increasing / stable / decreasing` | "Meeting load growing" |
| `meeting_hours_delta` | Hours/week change | "+4 hrs/week" |
| `one_on_one_consistency` | Std deviation of 1:1 intervals — low = consistent, high = erratic | Review signal for manager self-accountability |
| `recurring_meeting_growth` | Net change in recurring meetings this person attends | "Added to 3 new recurring meetings this quarter" |
| `attendance_in_declining_meetings` | Count of recurring meetings where team attendance dropped >20% but this person still attends | Signals someone stuck in dying meetings |

### 4.3 Performance Score (composite — for review ranking)

A single 0–100 score summarizing calendar-derived performance health:

```
performance_score = weighted average of:
  - focus_ratio (weight: 0.30) — normalized to 0-100 where 70%+ = 100
  - meeting_load_health (weight: 0.20) — inverse of hours, capped
  - one_on_one_coverage (weight: 0.20) — 1.0 if within 14-day cadence
  - trend_direction (weight: 0.15) — bonus for improving, penalty for declining
  - meeting_quality (weight: 0.15) — % of meetings with agendas and good attendance
```

**This score is NOT shown directly to anyone.** It's used by Workstation 3's review engine to prioritize which issues to highlight and by Workstation 5's recommendation engine to rank suggestions.

---

## 5. Team Analytics

Aggregated across all team members.

| Metric | Calculation |
|---|---|
| `total_meeting_spend` | Sum of all meeting costs in the period |
| `avg_meeting_hours_per_person` | Mean of individual meeting_hours |
| `avg_focus_ratio` | Mean of individual focus_ratios |
| `one_on_one_coverage` | % of direct reports with a 1:1 in the last 14 days |
| `top_cost_meetings` | Top 10 meetings by monthly cost |
| `low_roi_meetings` | Meetings where `attendance_rate < 60%` AND `monthly_cost > $500` |
| `team_meeting_load_distribution` | Per-person meeting hours for comparison (who's overloaded vs. underloaded) |
| `total_waste_cost` | Sum of `waste_cost` across all recurring meetings |

### Team Comparison Metrics (for reviews)

For each person, also compute their position relative to the team:

- `meeting_hours_vs_team_avg` — e.g., "42% above team average"
- `focus_ratio_vs_team_avg` — e.g., "15pp below team average"
- `percentile_meeting_load` — where they rank (useful for identifying outliers)

---

## 6. Flag Generation

Flags are binary alerts. Each flag has a severity level and a data-backed reason string.

| Flag | Trigger | Severity | Reason template |
|---|---|---|---|
| `burnout_risk` | meeting_hours > 25/week AND focus_ratio < 40% for 2+ consecutive weeks | HIGH | "{name} has averaged {hours} meeting hrs/week with only {ratio}% focus time over the last {weeks} weeks" |
| `no_1on1` | one_on_one_gap_days > 14 | MEDIUM | "No 1:1 with {name} in {days} days. Last one was on {date}" |
| `overloaded` | meeting_hours > team_avg × 1.3 | MEDIUM | "{name} is {pct}% above team average meeting load" |
| `declining_focus` | focus_ratio dropped >10pp over 4 weeks | HIGH | "{name}'s focus time dropped from {old}% to {new}% over the last month" |
| `stuck_in_dead_meetings` | attends 2+ recurring meetings with <50% team attendance | MEDIUM | "{name} still attends {count} recurring meetings that most of the team has stopped joining" |
| `back_to_back_chronic` | 3+ days/week with back-to-back streaks of 3+ meetings | HIGH | "{name} has back-to-back meeting blocks on {days} this week" |
| `healthy` | focus_ratio > 60% AND no other flags | LOW | "{name}'s calendar is well-balanced — {ratio}% focus time, regular 1:1s" |

**Every flag must include the data that triggered it.** Workstation 3 uses flag data directly in review narratives.

---

## 7. API Endpoints You Own

| Endpoint | Method | Returns | Consumers |
|---|---|---|---|
| `GET /api/analytics/person/{id}?period=quarter&start=&end=` | GET | Full person analytics including trends | WS3 (Review), WS4 (Dashboard) |
| `GET /api/analytics/person/{id}/history?weeks=12` | GET | Weekly time series of all metrics | WS4 (Sparklines), WS3 (Trend narratives) |
| `GET /api/analytics/team?period=month` | GET | Team-level aggregations | WS4 (Dashboard hero metrics) |
| `GET /api/analytics/team/comparison` | GET | Per-person metrics normalized against team avg | WS3 (Review comparisons) |
| `GET /api/analytics/flags/{member_id}` | GET | Active flags with severity and data | WS3 (Review), WS4 (Dashboard badges) |
| `GET /api/analytics/flags/team` | GET | All active flags across team | WS4 (Dashboard), WS5 (Recommendations) |
| `GET /api/cost/meeting/{id}` | GET | Single meeting cost breakdown | WS4 (Meeting detail view) |
| `GET /api/cost/recurring` | GET | All recurring meetings with monthly/annual costs and waste | WS4 (Cost chart), WS5 (Kill recommendations) |
| `GET /api/cost/team/summary?period=month` | GET | Total spend, per-person spend, top cost meetings | WS4 (Hero metric) |
| `GET /api/analytics/review-data/{member_id}?review_period=quarter` | GET | **Bundled payload** specifically shaped for review generation — includes all person analytics, trends, flags, team comparisons, 1:1 history, recurring meeting analysis | WS3 (Review Engine — primary consumer) |

### The Review Data Bundle

The `/api/analytics/review-data/{member_id}` endpoint is the most important one you build. It packages everything Workstation 3 needs into a single call:

```json
{
  "member": { "name, role, hourly_rate, ..." },
  "period": { "start, end, label (Q1 2026)" },
  "current_quarter": {
    "meeting_hours_per_week": 27,
    "focus_ratio": 0.35,
    "meeting_cost_total": 8400,
    "one_on_one_count": 6,
    "one_on_one_avg_interval_days": 10,
    "meetings_attended": 142,
    "decline_rate": 0.05,
    "agenda_less_recurring_count": 3,
    "back_to_back_days_per_week": 3.2
  },
  "previous_quarter": { "...same shape..." },
  "trends": {
    "focus_ratio_delta": -17,
    "focus_ratio_direction": "declining",
    "meeting_hours_delta": 8,
    "meeting_hours_direction": "increasing",
    "recurring_meeting_net_change": 3
  },
  "team_comparison": {
    "team_avg_meeting_hours": 19,
    "member_vs_avg_pct": 42,
    "team_avg_focus_ratio": 0.52,
    "member_focus_rank": "5/5"
  },
  "flags": [
    { "type": "burnout_risk", "severity": "HIGH", "reason": "...", "data": {} },
    { "type": "stuck_in_dead_meetings", "severity": "MEDIUM", "reason": "...", "data": {} }
  ],
  "recurring_meetings_analysis": [
    {
      "title": "Wednesday Product Sync",
      "monthly_cost": 2400,
      "attendance_rate": 0.47,
      "member_attendance_rate": 1.0,
      "trend": "declining"
    }
  ],
  "one_on_one_history": [
    { "date": "2026-01-15", "duration_minutes": 30 },
    { "date": "2026-01-28", "duration_minutes": 25 }
  ],
  "performance_score": 38,
  "weekly_history": [
    { "week": "2026-W01", "meeting_hours": 22, "focus_ratio": 0.45 },
    { "week": "2026-W02", "meeting_hours": 24, "focus_ratio": 0.42 }
  ]
}
```

---

## 8. Performance Considerations

- All analytics must compute from cached SQLite data (populated by WS1), never from live MCP calls
- The review data bundle for one person must return in <500ms
- Team analytics must return in <1s
- Pre-compute weekly aggregations on sync (don't recompute from raw events on every request)
- Store computed weekly snapshots in a `weekly_analytics` table

---

## 9. Handoff Contract

You are DONE when:

- [ ] Cost engine correctly calculates per-meeting, monthly, annual, and waste costs
- [ ] Person analytics returns all core + trend metrics for any time period
- [ ] Team analytics returns aggregations and per-person comparisons
- [ ] Flag generation fires correctly for all defined triggers with data-backed reason strings
- [ ] `/api/analytics/review-data/{id}` returns the complete bundled payload
- [ ] Weekly history endpoint returns 12+ weeks of time series data
- [ ] Performance score computes a stable 0–100 composite
- [ ] All endpoints respond within performance targets

---

## 10. What You Do NOT Build

- MCP connections or data fetching (Workstation 1)
- Review narrative text or Claude prompts (Workstation 3)
- Any UI (Workstation 4)
- Chat interface or natural language processing (Workstation 5)

You compute numbers. Accurate numbers. Fast numbers. That's your job.
