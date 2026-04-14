# Workstation 4 — Frontend Dashboard & Review UI

**Owner:** Developer 4
**Focus:** The entire user interface — team dashboard, performance review panels, charts, calendar views, and visual design. You make the data visible and the reviews readable.

---

## 1. Mission

You build everything the manager sees and touches — except the chat panel (that's Workstation 5). Your primary focus for this version is the **Performance Review UI**: the screens where a manager prepares, reads, and acts on data-driven reviews powered by MCP-sourced calendar data.

The dashboard is the entry point. The review panel is the destination.

---

## 2. Tech Stack

| Tool | Purpose |
|---|---|
| Next.js 14 (App Router) | Framework, SSR for initial dashboard load |
| React | Component architecture |
| Tailwind CSS | Styling |
| Recharts or Tremor | Charts and sparklines |
| Framer Motion | Transitions and loading states |

---

## 3. Dependencies (APIs you consume)

| API | Source | What you render |
|---|---|---|
| `GET /api/cost/team/summary` | WS2 | Hero metric (monthly spend) |
| `GET /api/analytics/team` | WS2 | Team averages, aggregations |
| `GET /api/analytics/person/{id}` | WS2 | Per-person cards |
| `GET /api/analytics/person/{id}/history` | WS2 | Sparklines, trend charts |
| `GET /api/analytics/flags/team` | WS2 | Badge alerts on cards |
| `GET /api/cost/recurring` | WS2 | Meeting cost chart |
| `POST /api/review/generate/{id}` | WS3 | Full performance review brief |
| `POST /api/review/snapshot/{id}` | WS3 | Quick review card |
| `POST /api/review/team-summary` | WS3 | Team review preparation view |
| `GET /api/review/history/{id}` | WS3 | Past reviews |
| `GET /api/sync/status` | WS1 | Data freshness indicator |
| `GET /api/team/members` | WS1 | Team roster |

The chat panel is built by Workstation 5 and embedded as a component. You provide the layout slot for it.

---

## 4. Page Structure

```
/                         → Team Dashboard (home)
/reviews                  → Review Center (team review prep)
/reviews/{member_id}      → Individual Review Brief
/meetings                 → Meeting Cost Explorer
/calendar                 → Calendar View (visual schedule)
```

---

## 5. Screen Specifications

### 5.1 Team Dashboard (`/`)

The first screen the manager sees. Designed around the question: **"How is my team spending time, and what should I do about it?"**

**Layout:**

```
┌─────────────────────────────────────────────────────────────┐
│  HEADER: Orquesta — [Team Name] — April 2026               │
│  MCP Status: ● Connected (last sync: 2 min ago)            │
├──────────────────────────────────┬──────────────────────────┤
│                                  │                          │
│  HERO METRIC                     │  REVIEW CENTER           │
│  $14,200                         │  (quick access)          │
│  Monthly Meeting Spend           │                          │
│  ▲ 8% vs last month             │  ⚠ 2 reviews due         │
│                                  │  Ana — At Risk           │
│  SECONDARY METRICS ROW           │  Carlos — Needs Attn     │
│  ┌──────┐ ┌──────┐ ┌──────┐     │                          │
│  │Focus │ │1:1   │ │Waste │     │  [Prepare Reviews →]     │
│  │ 48%  │ │ 75%  │ │$1.2k │     │                          │
│  └──────┘ └──────┘ └──────┘     │                          │
│                                  ├──────────────────────────┤
│  TEAM MEMBER CARDS               │                          │
│  ┌─────────────────────────┐     │  CHAT PANEL              │
│  │ Ana Oliveira  SR ENG    │     │  (Workstation 5 slot)    │
│  │ 27 hrs  35% focus  ⚠🔴 │     │                          │
│  │ "At Risk — burnout"     │     │                          │
│  │ [View Review]           │     │                          │
│  └─────────────────────────┘     │                          │
│  ┌─────────────────────────┐     │                          │
│  │ Diego Ramirez  MID ENG  │     │                          │
│  │ 15 hrs  62% focus  ✓🟢 │     │                          │
│  │ "Healthy"               │     │                          │
│  └─────────────────────────┘     │                          │
│  ...                             │                          │
│                                  │                          │
├──────────────────────────────────┤                          │
│  MEETING COST CHART              │                          │
│  (bar chart: top meetings by $)  │                          │
│                                  │                          │
│  FOCUS TIME CHART                │                          │
│  (sparklines per person, 8wk)    │                          │
└──────────────────────────────────┴──────────────────────────┘
```

**Component Details:**

**Hero Metric Card**
- Large number: `$14,200` — monthly meeting spend
- Subtitle: trend vs previous month with directional arrow
- Color: neutral (it's informational, not good/bad)

**Secondary Metrics Row** — three small cards:
- Average Focus Ratio — color-coded (green >50%, yellow 35-50%, red <35%)
- 1:1 Coverage — percentage of reports with recent 1:1s
- Monthly Waste — sum of waste_cost from low-ROI meetings

**Team Member Cards** — one per person, sorted by urgency (At Risk first):
- Name, role, timezone
- Meeting hours/week
- Focus ratio with color indicator
- Status badge: Healthy (green), Needs Attention (yellow), At Risk (red)
- Active flags as small badges
- Sparkline: 8-week meeting hours trend
- **"View Review" button** — navigates to `/reviews/{id}`

**MCP Status Indicator** — top right:
- Green dot + "Connected" when last sync < 15 minutes
- Yellow dot + "Stale" when 15-60 minutes
- Red dot + "Disconnected" when > 60 minutes or sync failed
- Clicking opens sync details panel

**Review Center Quick Access** — right sidebar above chat:
- Shows count of reviews due (based on `next_review_date`)
- Lists members needing attention, sorted by severity
- "Prepare Reviews" button navigates to `/reviews`

### 5.2 Review Center (`/reviews`)

The team-wide review preparation screen.

**Layout:**

```
┌──────────────────────────────────────────────────────────────┐
│  REVIEW CENTER — Q1 2026 Performance Reviews                 │
│  [Generate Team Summary]                                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  TEAM SUMMARY PANEL (collapsible, loads on button click)     │
│  - Systemic patterns                                         │
│  - Manager self-assessment                                   │
│  - Total optimization potential                              │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  MEMBER REVIEW CARDS (sorted by urgency)                     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ 🔴 Ana Oliveira — At Risk — Score: 38/100            │    │
│  │ Focus: 35% (↓17pp) | Meetings: 27h/wk (↑42% vs avg) │    │
│  │ Flags: burnout_risk, stuck_in_dead_meetings           │    │
│  │ [Generate Full Review]  [Quick Snapshot]               │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ 🟡 Carlos Mendez — Needs Attention — Score: 55/100   │    │
│  │ Focus: 48% (stable) | 1:1 gap: 18 days               │    │
│  │ Flags: no_1on1                                        │    │
│  │ [Generate Full Review]  [Quick Snapshot]               │    │
│  └──────────────────────────────────────────────────────┘    │
│  ...                                                         │
└──────────────────────────────────────────────────────────────┘
```

### 5.3 Individual Review Brief (`/reviews/{member_id}`)

The most important screen in the app. This is where the manager reads and uses the performance review.

**Layout:**

```
┌──────────────────────────────────────────────────────────────┐
│  ← Back to Reviews                                           │
│                                                              │
│  REVIEW HEADER                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Ana Oliveira — Senior Engineer                        │    │
│  │ Q1 2026 Performance Review Brief                      │    │
│  │ 🔴 At Risk          Generated: Apr 14, 2026, 2:30pm  │    │
│  │ Data sources: Google Calendar MCP, Google Drive MCP   │    │
│  │ [Regenerate]  [Export PDF]  [Share]                    │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  METRICS STRIP (visual summary bar)                          │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐             │
│  │Focus │ │Mtg   │ │1:1   │ │Flags │ │Cost  │             │
│  │ 35%  │ │27h/w │ │10d   │ │  2   │ │$8.4k │             │
│  │  🔴  │ │  🔴  │ │  🟢  │ │  🔴  │ │  🔴  │             │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘             │
│                                                              │
│  EXECUTIVE SUMMARY                                           │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Claude-generated narrative...                         │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  SECTIONS (collapsible, all expanded by default)             │
│  ┌─ Time Allocation ──────────────────────────────────┐     │
│  │ Narrative + [CHART: meeting hours over 12 weeks]     │     │
│  └─────────────────────────────────────────────────────┘     │
│  ┌─ Focus Time Analysis ──────────────────────────────┐     │
│  │ Narrative + [CHART: focus ratio trend with team avg] │     │
│  └─────────────────────────────────────────────────────┘     │
│  ┌─ Meeting Quality ──────────────────────────────────┐     │
│  │ Narrative + [TABLE: recurring meetings with costs]   │     │
│  └─────────────────────────────────────────────────────┘     │
│  ┌─ Coaching Relationship ────────────────────────────┐     │
│  │ Narrative + [TIMELINE: 1:1 dates over quarter]       │     │
│  └─────────────────────────────────────────────────────┘     │
│  ┌─ Risk Flags ───────────────────────────────────────┐     │
│  │ Flag cards with severity and evidence                │     │
│  └─────────────────────────────────────────────────────┘     │
│  ┌─ Strengths ────────────────────────────────────────┐     │
│  │ Positive patterns observed                           │     │
│  └─────────────────────────────────────────────────────┘     │
│  ┌─ Recommendations ─────────────────────────────────┐     │
│  │ Numbered actions with [Implement] buttons            │     │
│  └─────────────────────────────────────────────────────┘     │
│  ┌─ Cost Context ─────────────────────────────────────┐     │
│  │ Dollar impact summary                                │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                              │
│  DATA SCOPE NOTE                                             │
│  "Based on Google Calendar and Drive data via MCP..."        │
│                                                              │
│  [Export as PDF]  [Prepare Next Review: Carlos →]            │
└──────────────────────────────────────────────────────────────┘
```

**Chart Details for Review Sections:**

- **Time Allocation Chart:** Stacked area chart — meeting hours vs focus hours, weekly over 12 weeks. Team average as dotted line overlay.
- **Focus Ratio Trend:** Line chart, person vs team average, with the quarter boundary marked.
- **Recurring Meetings Table:** Sortable table columns: Meeting Name | Monthly Cost | Attendance Rate | Trend | This Person's Attendance | [Action].
- **1:1 Timeline:** Horizontal dot timeline showing each 1:1 date. Gaps >14 days highlighted in red. Target cadence shown as grey zone.

### 5.4 Meeting Cost Explorer (`/meetings`)

Secondary screen. Sortable/filterable list of all meetings with costs.

- Table: Meeting Name | Type | Attendees | Duration | Per-Session Cost | Monthly Cost | Attendance Rate | Agenda?
- Filters: by person, by type, by cost threshold, by date range
- Sort: by cost, by attendance, by frequency
- Click any meeting → detail modal with attendee breakdown

### 5.5 Calendar View (`/calendar`)

Week view showing scheduled events for the team. Used during the scheduling demo act.

- Color-coded by category (1:1 = blue, team sync = purple, external = orange, focus block = green stripes)
- Toggle between team members
- Events show cost on hover
- Created events (from chat) animate in with a highlight

---

## 6. Component Library

Build these as reusable components:

| Component | Props | Used in |
|---|---|---|
| `MetricCard` | label, value, trend, color, size | Dashboard, Review |
| `StatusBadge` | status (healthy/attention/risk) | Dashboard cards, Review header |
| `FlagBadge` | flag type, severity | Dashboard cards, Review flags |
| `SparklineChart` | data[], color | Dashboard member cards |
| `TrendChart` | data[], comparator[], labels | Review sections |
| `CostBar` | meetings[], maxCost | Meeting explorer |
| `ReviewSection` | title, content (markdown), chart?, collapsible | Review brief |
| `MemberCard` | member analytics data | Dashboard, Review center |
| `MCPStatusDot` | syncStatus | Header |
| `ActionButton` | label, onClick, variant | Review recommendations |
| `ChatSlot` | — | Layout container for WS5's chat |

---

## 7. Loading & Error States

- **Dashboard load:** Skeleton cards with pulse animation. Load metrics first (fast), then charts (slower).
- **Review generation:** Progress indicator with stages: "Pulling analytics..." → "Analyzing patterns..." → "Generating review..." (stages map to WS2 call → WS3 call)
- **MCP disconnected:** Banner at top: "Calendar data may be stale. Last synced: {time}. [Retry Sync]"
- **Review generation failed:** Show cached review if available with "(cached)" label. If no cache: "Review generation temporarily unavailable. [Retry] or use the chat panel for a quick snapshot."

---

## 8. Responsive Design

- Primary: Desktop (1440px+) — full layout with side chat panel
- Secondary: Tablet (768-1439px) — chat panel becomes bottom sheet
- Tertiary: Mobile (< 768px) — single column, chat accessible via floating button

---

## 9. Export Functionality

- **Export Review as PDF:** Use browser print-to-PDF or a server-side PDF generation endpoint (coordinate with WS3)
- **Export Dashboard as PNG:** Screenshot of the dashboard summary area
- **Copy Review to Clipboard:** Markdown-formatted text for pasting into Google Docs, Notion, etc.

---

## 10. Handoff Contract

You are DONE when:

- [ ] Dashboard loads with all metrics, cards, charts, and badges from WS2 APIs
- [ ] Review Center lists all team members sorted by urgency with correct badges
- [ ] Individual Review page renders all 10 sections with charts and narratives from WS3
- [ ] Charts render accurately from WS2 history data (sparklines, trend lines, comparisons)
- [ ] "Generate Review" button triggers WS3 API and shows progress states
- [ ] MCP status indicator reflects real sync status from WS1
- [ ] "View Review" on dashboard cards navigates to the correct review page
- [ ] Chat slot renders WS5's component correctly at all breakpoints
- [ ] Loading, error, and empty states are all handled
- [ ] Demo flow (Acts 1-5) can be performed without any UI issues

---

## 11. What You Do NOT Build

- Chat panel or chat UI (Workstation 5 — you provide the layout slot)
- API endpoints (Workstations 1, 2, 3)
- Cost calculations or analytics (Workstation 2)
- Review narrative generation or Claude prompts (Workstation 3)

You render what others compute. Make it beautiful, fast, and clear.
