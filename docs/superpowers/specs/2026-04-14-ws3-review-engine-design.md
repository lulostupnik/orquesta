# WS3 — Performance Review Engine: Design Spec

**Date:** 2026-04-14
**Workstation:** 3 — Performance Review Engine
**Owner:** Developer 3
**Status:** Approved, ready for implementation

---

## 1. Overview

WS3 is a standalone Express + TypeScript service (port 3003) that transforms analytics data from WS2 into structured, AI-generated performance review narratives via the Claude API. It owns all prompt engineering for reviews and exposes 6 HTTP endpoints consumed by WS4 (Frontend) and WS5 (Chat).

**Core data flow:**
```
WS2 analytics bundle → analyticsClient → generator → Claude API → calibration → SQLite → HTTP response
```

---

## 2. Tech Stack

| Concern | Choice |
|---|---|
| Runtime | Node.js + TypeScript |
| Framework | Express |
| Claude SDK | `@anthropic-ai/sdk` |
| Database | SQLite via `better-sqlite3` |
| Port | 3003 |
| Mode switch | `MODE=demo` env var |

---

## 3. File Structure

```
ws3-review-engine/
├── src/
│   ├── index.ts                  # Express app setup, port 3003
│   ├── routes/
│   │   └── review.ts             # All 6 route handlers (thin — delegate immediately)
│   ├── services/
│   │   ├── generator.ts          # Core: calls Claude, applies calibration, stores result
│   │   ├── analyticsClient.ts    # WS2 client: real HTTP or fixture fallback
│   │   └── calibration.ts        # Badge assignment, consistency checks, sensitivity rules
│   ├── db/
│   │   └── repository.ts         # SQLite: save/load reviews, commitments
│   ├── prompts/
│   │   ├── fullReview.ts         # System prompt + user prompt builder for full review
│   │   ├── snapshot.ts           # System + user prompt for quick snapshot
│   │   └── teamSummary.ts        # System + user prompt for team summary
│   ├── scripts/
│   │   └── pregenerate.ts        # Pre-generates and caches reviews for all 5 demo members
│   └── types.ts                  # Shared TypeScript interfaces
├── fixtures/
│   ├── ana-oliveira.json         # WS2 bundle (demo "At Risk" — primary demo member)
│   ├── carlos-mendez.json        # WS2 bundle (demo "Needs Attention" — 1:1 gap)
│   ├── diego-ramirez.json        # WS2 bundle (demo "Healthy")
│   ├── sofia-chen.json           # WS2 bundle (varied flags)
│   └── marcos-silva.json         # WS2 bundle (varied flags)
├── data/
│   └── reviews.db                # SQLite database (gitignored)
├── package.json
├── tsconfig.json
└── .env                          # ANTHROPIC_API_KEY, MODE, WS2_URL, PORT
```

---

## 4. Data Models

### 4.1 Input — WS2 Analytics Bundle (`ReviewBundle`)

```typescript
interface ReviewBundle {
  member: {
    id: string;
    name: string;
    role: string;
    hourly_rate: number;
  };
  period: { start: string; end: string; label: string };
  current_quarter: {
    meeting_hours_per_week: number;
    focus_ratio: number;
    meeting_cost_total: number;
    one_on_one_count: number;
    one_on_one_avg_interval_days: number;
    one_on_one_gap_days: number;
    meetings_attended: number;
    decline_rate: number;
    agenda_less_recurring_count: number;
    back_to_back_days_per_week: number;
  };
  previous_quarter: { /* same shape as current_quarter */ };
  trends: {
    focus_ratio_delta: number;
    focus_ratio_direction: "improving" | "stable" | "declining";
    meeting_hours_delta: number;
    meeting_hours_direction: "increasing" | "stable" | "decreasing";
    recurring_meeting_net_change: number;
  };
  team_comparison: {
    team_avg_meeting_hours: number;
    member_vs_avg_pct: number;
    team_avg_focus_ratio: number;
    member_focus_rank: string;
  };
  flags: Array<{
    type: string;
    severity: "HIGH" | "MEDIUM" | "LOW";
    reason: string;
    data: object;
  }>;
  recurring_meetings_analysis: Array<{
    title: string;
    monthly_cost: number;
    attendance_rate: number;
    member_attendance_rate: number;
    trend: "declining" | "stable" | "growing";
  }>;
  one_on_one_history: Array<{ date: string; duration_minutes: number }>;
  performance_score: number;
  weekly_history: Array<{ week: string; meeting_hours: number; focus_ratio: number }>;
}
```

### 4.2 Output — Review Result (`ReviewResult`)

```typescript
interface ReviewResult {
  review_id: string;
  member_id: string;
  member_name: string;
  period: string;
  generated_at: string;
  status_badge: "healthy" | "needs_attention" | "at_risk";
  performance_score: number;
  sections: {
    executive_summary: string;
    time_allocation: string;
    focus_analysis: string;
    meeting_quality: string;
    coaching_relationship: string;
    risk_flags: string;
    strengths: string;
    recommendations: string;
    cost_context: string;
    data_scope_note: string;
  };
  raw_metrics: Partial<ReviewBundle>;
  data_sources: string[];
  model: string;
  generation_time_ms: number;
  cached?: boolean;
}
```

### 4.3 SQLite Schema

```sql
CREATE TABLE reviews (
  review_id    TEXT PRIMARY KEY,
  member_id    TEXT NOT NULL,
  period       TEXT NOT NULL,
  generated_at TEXT NOT NULL,
  result_json  TEXT NOT NULL,
  is_cached    INTEGER DEFAULT 0
);
CREATE INDEX idx_reviews_member ON reviews(member_id, generated_at DESC);

CREATE TABLE commitments (
  id           TEXT PRIMARY KEY,
  review_id    TEXT NOT NULL,
  member_id    TEXT NOT NULL,
  payload_json TEXT NOT NULL,
  created_at   TEXT NOT NULL
);
```

---

## 5. API Endpoints

| Endpoint | Method | Handler behavior |
|---|---|---|
| `/api/review/generate/:id` | POST | Check 24hr cache → call `generator.generateFull()` → return `ReviewResult` |
| `/api/review/snapshot/:id` | POST | Call `generator.generateSnapshot()` → return compact result (not stored) |
| `/api/review/team-summary` | POST | Fetch all bundles → one Claude call → return ranked team overview |
| `/api/review/history/:id` | GET | `repository.getHistory(id)` → array of past `ReviewResult` |
| `/api/review/commit` | POST | Validate body → `repository.saveCommitment()` → return saved commitment |
| `/api/review/commitments/:id` | GET | `repository.getCommitments(id)` → active commitments array |

### Request body for generate

```typescript
interface GenerateRequest {
  review_period?: string;      // default: current quarter
  include_sections?: string[]; // default: ["all"]
  comparison_mode?: "team_average" | "previous_quarter" | "both";
  force?: boolean;             // bypass 24hr cache
}
```

### Error responses

| Condition | Status | Body |
|---|---|---|
| WS2 unreachable + no fixture | 503 | `{ error: "Analytics data unavailable" }` |
| Claude timeout/error | 503 | `{ error: "Review generation failed", cached_available: boolean }` |
| Member not found (demo mode) | 404 | `{ error: "Member not found" }` |
| Invalid request body | 400 | `{ error: string }` |

---

## 6. Claude Integration

### 6.1 API Call Structure

Uses prompt caching on the system prompt (large, static — meaningful latency/cost savings on demo day):

```typescript
const response = await anthropic.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 2048,
  system: [
    {
      type: "text",
      text: FULL_REVIEW_SYSTEM_PROMPT,
      cache_control: { type: "ephemeral" }
    }
  ],
  messages: [
    { role: "user", content: buildUserPrompt(bundle) }
  ]
});
```

### 6.2 Prompt Files

- `prompts/fullReview.ts` — exports `FULL_REVIEW_SYSTEM_PROMPT` (verbatim from WS3 spec §4.1) and `buildUserPrompt(bundle: ReviewBundle): string`
- `prompts/snapshot.ts` — shorter system prompt, returns compact JSON (status_badge, one_liner, key_metrics, top_flag, top_recommendation)
- `prompts/teamSummary.ts` — team-analysis system prompt from spec §4.3

`buildUserPrompt` serializes the bundle compactly (no pretty-print) to stay under 4,000 tokens. If estimated length exceeds 16,000 chars: trim `weekly_history` to last 8 weeks and `one_on_one_history` to last 10 entries.

### 6.3 Timeout

10-second hard timeout on every Claude call. On timeout:
1. Check SQLite for any cached review for this member
2. If found → return it with `{ cached: true }`
3. If not found → return 503

---

## 7. Calibration Layer (`calibration.ts`)

Applied to every Claude response before storage and return.

### 7.1 Badge Assignment (deterministic)

```
score >= 70 AND no HIGH flags  → "healthy"
score 40–69 OR any MEDIUM flag → "needs_attention"
score < 40  OR any HIGH flag   → "at_risk"
```

Badge is assigned by `calibration.ts` from the `performance_score` and `flags` in the bundle — **not** from Claude's output.

### 7.2 Consistency Checks

Run after Claude returns:

1. `focus_ratio` trend is improving but narrative contains "declining" → trigger one retry. If retry also fails → serve cached fallback.
2. `one_on_one_gap_days < 10` but `coaching_relationship` section flags 1:1 gaps → strip the gap commentary from that section.
3. Badge is `at_risk` but no flags are active in bundle → demote to `needs_attention`.
4. Dollar figures extracted from narrative via regex → compared against bundle values ±5% → log warning on drift (do not block response).

### 7.3 Sensitivity Rules

Post-process all narrative string fields:

- Banned terms (`underperforming`, `failing`, `problematic`, `poor performer`) → replaced with structural framing (`constrained by`, `impacted by`, `could benefit from`)
- `strengths` section must be non-empty — if Claude returns empty, inject a default sentence derived from the best metric in the bundle
- 1:1 gap commentary must attribute the gap to cadence/scheduling, not to the report personally

---

## 8. Demo Mode & Pre-generation

### 8.1 MODE env var

`analyticsClient.ts` is the only file that reads `MODE`:

- `MODE=demo` → load from `fixtures/<member-id>.json`, return synchronously
- `MODE=live` → HTTP call to `WS2_URL/api/analytics/review-data/:id`

No other code changes. All downstream code is unaware of the mode.

### 8.2 Fixture: Ana Oliveira (primary demo member)

Matches spec §8 exactly:

```json
{
  "member": { "id": "ana-oliveira", "name": "Ana Oliveira", "role": "senior_eng", "hourly_rate": 75 },
  "current_quarter": {
    "meeting_hours_per_week": 27, "focus_ratio": 0.35,
    "meeting_cost_total": 8400, "one_on_one_avg_interval_days": 10,
    "back_to_back_days_per_week": 3.2, "agenda_less_recurring_count": 3
  },
  "trends": { "focus_ratio_delta": -17, "focus_ratio_direction": "declining",
               "meeting_hours_delta": 8, "meeting_hours_direction": "increasing" },
  "team_comparison": { "team_avg_meeting_hours": 19, "member_vs_avg_pct": 42,
                        "team_avg_focus_ratio": 0.52, "member_focus_rank": "5/5" },
  "flags": [
    { "type": "burnout_risk", "severity": "HIGH", "reason": "27 hrs/week with 35% focus time" },
    { "type": "stuck_in_dead_meetings", "severity": "MEDIUM", "reason": "3 recurring meetings <60% attendance" }
  ],
  "performance_score": 38
}
```

The other 4 fixtures cover the full spectrum: one healthy (Diego), one needs_attention with 1:1 gap (Carlos), two with varied flag patterns (Sofia, Marcos).

### 8.3 Pre-generation Script

```
npx ts-node src/scripts/pregenerate.ts
```

Run once before demo. For each fixture: loads bundle → calls `generateFull()` → stores with `is_cached = 1`. Logs summary per member. After pre-generation, all `/api/review/generate/:id` calls return from SQLite in <50ms with no Claude call required.

Force-bypass available via `{ force: true }` in request body — useful during demo to show live generation if internet is reliable.

---

## 9. Complexity Assessment

This is a **medium-high complexity** workstation for a hackathon:

| Area | Complexity | Notes |
|---|---|---|
| Express boilerplate + routes | Low | Straightforward |
| SQLite setup + repository | Low | `better-sqlite3` is simple |
| `analyticsClient.ts` (mode switch) | Low | ~50 lines |
| Prompt engineering (3 prompts) | Medium | System prompts are specified in WS3 spec — mostly copy + parameterize |
| Claude API integration + caching | Medium | Prompt caching adds a small API surface area |
| Calibration layer | Medium-High | Consistency checks + sensitivity regex + badge logic — lots of edge cases |
| Fixture JSON files (5 members) | Medium | Must be internally consistent and match spec numbers exactly |
| Pre-generation script | Low | ~40 lines, calls existing generator |
| **Total estimated effort** | **~1 day for one developer** | Calibration is the risk area — budget extra time |

**Risk:** The calibration consistency checks are the most likely source of bugs. The regex-based dollar figure comparison and the "narrative says X but data says Y" detection are fragile. Mitigation: log failures without blocking, fix edge cases after first end-to-end test.

---

## 10. Handoff Contract

Done when:
- [ ] `POST /api/review/generate/:id` returns a full multi-section review with accurate data citations
- [ ] `POST /api/review/snapshot/:id` returns in <3 seconds
- [ ] `POST /api/review/team-summary` returns a ranked team overview
- [ ] Status badges correctly assigned from score + flags
- [ ] Consistency checks catch and correct Claude mismatches
- [ ] Reviews stored and retrievable from history
- [ ] Pre-generated cached reviews exist for all 5 demo members
- [ ] Dollar figures in narratives match data ±5%
- [ ] Sensitivity rules enforced (no negative person-language)
- [ ] Data scope note reflects MCP sources used
- [ ] End-to-end generation completes in <10 seconds
- [ ] `MODE=demo` serves fixture data with no WS2 dependency
