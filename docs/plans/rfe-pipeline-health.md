# RFE Pipeline Health — Implementation Plan

## Overview

Extend the **RFE Review** view in the AI Impact module to surface pipeline friction signals alongside existing adoption metrics. Today Org Pulse tracks the upside of the rfe-creator pipeline (how many RFEs were AI-created or AI-revised) but does not aggregate failure and human-overhead labels. This plan adds a **Pipeline Health** section directly below the current adoption metrics row so PMs can answer: *how often does the automation fail, and where?*

**Scope:** RFE Review UI only. No new nav tab. No new Jira fetch — labels are already stored on every cached RFE issue.

**Related docs:**
- `docs/ai-impact-implementation-plan.md` — assessment visualization (rubric pass/fail; separate from pipeline labels)
- `modules/ai-impact/server/jira/rfe-fetcher.js` — existing RFE ingest
- `modules/ai-impact/server/metrics.js` — existing adoption aggregation

---

## Background

The rfe-creator pipeline stamps Jira labels on RHAIRFE tickets to indicate what the automation did. Org Pulse already fetches the full `labels` array for every issue and persists it in `data/ai-impact/rfe-data.json`. However, `metrics.js` only classifies `aiInvolvement` from two labels:

| Label | Used today? |
|-------|-------------|
| `rfe-creator-auto-created` | Yes — adoption % |
| `rfe-creator-auto-revised` | Yes — revised count |
| `rfe-creator-needs-attention` | **No** — stored on issue, not aggregated |
| `rfe-creator-feasibility-fail` | **No** |
| `rfe-creator-feasibility-unknown` | **No** |
| `rfe-creator-autofix-rubric-pass` | **No** |
| `rfe-creator-feasibility-pass` | **No** |

Because these friction labels are not aggregated, no one can currently answer:

- What % of AI pipeline runs require manual human cleanup?
- Is pipeline quality getting better or worse over time?
- Which PMs hit the most automation blockers?

**Note:** Rubric pass/fail (`PASS`/`FAIL` from assessments) is a separate signal already surfaced in RFE list filters and detail panels. Pipeline Health focuses on **Jira pipeline labels**, not assessment scores.

---

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| UI placement | New **Pipeline Health** section below `MetricsRow` in RFE Review | Keeps ADLC phase nav unchanged; separates adoption volume from friction quality |
| Data source | Existing `rfe-data.json` issue list (`labels`, `creator`, `creatorDisplayName`, `created`) | No backend fetch changes; aligns with display-layer architecture |
| Aggregation location | Server-side in `metrics.js` (returned with existing RFE metrics API response) | Consistent with adoption metrics; enables unit tests; frontend stays thin |
| Time window | Same window as adoption metrics (`week` / `month` / `3months`, filtered by `issue.created`) | Reuses existing selector; predictable behavior |
| Denominator for rates | AI-touched RFEs in window (`aiInvolvement !== 'none'`) | Avoids diluting rates with tickets the pipeline never ran on |
| Topic breakdown | **Phase 1:** group by `creatorDisplayName`. **Phase 2 (optional):** keyword/component grouping if upstream data added | RFE fetch does not include Jira components today |
| Label config | Hardcode label strings in `metrics.js` initially; optionally move to `config.js` later | Matches existing pattern for `createdLabel` / `revisedLabel` |

---

## Pipeline Health Metrics

### Summary tiles (Pipeline Health row)

Four primary friction signals for the selected time window:

| Metric | Label(s) | Definition |
|--------|----------|------------|
| **Human Cleanup Rate** | `rfe-creator-needs-attention` | % of AI-touched RFEs in window with this label |
| **Feasibility Fail Rate** | `rfe-creator-feasibility-fail` | % of AI-touched RFEs flagged infeasible |
| **Feasibility Unknown Rate** | `rfe-creator-feasibility-unknown` | % of AI-touched RFEs where feasibility could not be determined |
| **Rubric Pass Rate** | `rfe-creator-autofix-rubric-pass` | % of AI-touched RFEs that passed automated rubric scoring |

Each tile should show:
- Current window value (%)
- Change vs prior period (pp delta, same pattern as `createdChange`)
- Optional trend direction (`growing` / `stable` / `declining`) for friction metrics where **lower is better**

### Trend chart (optional, Phase 2)

Extend `TrendCharts` or add a small line chart showing weekly:
- Needs-attention rate
- Feasibility fail + unknown rate combined

Bucket by `issue.created` (same limitation as adoption trends — label transition timestamps are not used unless changelog is parsed later).

### Breakdown tables

**By PM** — for AI-touched RFEs in window, group by `creatorDisplayName`:

| PM | AI RFEs | Needs Attention | Feasibility Fail | Unknown | Rubric Pass |
|----|---------|-----------------|------------------|---------|-------------|

Sort by highest friction rate by default. Intended to highlight training or onboarding gaps, not blame individuals.

**By topic (deferred)** — requires a stable grouping field (Jira component, linked feature area, or assessment anti-pattern). Not in Phase 1.

---

## Architecture

```
Jira (RHAIRFE)
      │
      ▼
rfe-fetcher.js          ← already fetches labels[]
      │
      ▼
rfe-data.json           ← cached issue list
      │
      ▼
metrics.js              ← ADD computePipelineHealthMetrics()
      │                    returns pipelineHealth alongside metrics/trendData
      ▼
GET /api/modules/ai-impact/rfe-data
      │
      ▼
RFEReviewView.vue
      │
      ▼
PhaseContent.vue
      ├─ MetricsRow.vue           (existing adoption tiles)
      └─ PipelineHealthRow.vue    (NEW friction tiles)
          PipelineHealthBreakdown.vue (NEW PM table, collapsible)
```

---

## Phased Implementation

```
Phase 1: Server aggregation + summary tiles
    Unit tests for label counting. Deployable without UI polish.

Phase 2: PM breakdown table + list filters
    Clickable rows filter RFE list to friction label.

Phase 3 (optional): Trend chart + topic grouping
    Requires design input on topic dimension.
```

Each phase is independently deployable and backward-compatible.

---

## Phase 1: Server Aggregation + Summary Tiles

### 1. Extend `metrics.js`

**File:** `modules/ai-impact/server/metrics.js`

Add label constants and a classifier:

```js
const PIPELINE_LABELS = {
  needsAttention: 'rfe-creator-needs-attention',
  feasibilityFail: 'rfe-creator-feasibility-fail',
  feasibilityUnknown: 'rfe-creator-feasibility-unknown',
  rubricPass: 'rfe-creator-autofix-rubric-pass',
  feasibilityPass: 'rfe-creator-feasibility-pass'
};

function classifyPipelineFriction(labels) {
  const set = new Set(labels || []);
  return {
    needsAttention: set.has(PIPELINE_LABELS.needsAttention),
    feasibilityFail: set.has(PIPELINE_LABELS.feasibilityFail),
    feasibilityUnknown: set.has(PIPELINE_LABELS.feasibilityUnknown),
    rubricPass: set.has(PIPELINE_LABELS.rubricPass),
    feasibilityPass: set.has(PIPELINE_LABELS.feasibilityPass)
  };
}
```

Add `computePipelineHealthMetrics(issues, timeWindow, config)`:
- Filter issues to current and prior windows (same cutoffs as `computeMetrics`)
- Denominator: issues where `aiInvolvement !== 'none'`
- Compute counts and percentages for each friction label
- Compute pp change vs prior period for each rate
- Return `{ summary, priorSummary, trend }` where lower friction = improving trend

Wire into `computeAllMetrics()` return value:

```js
return {
  metrics: computeMetrics(...),
  trendData: buildTrendData(...),
  breakdown: buildBreakdownData(...),
  pipelineHealth: computePipelineHealthMetrics(issues, timeWindow, config)  // NEW
};
```

### 2. Unit tests

**File:** `modules/ai-impact/__tests__/server/pipeline-health.test.js` (NEW)

Cover:
- Rate calculation with mixed labels
- Denominator excludes `aiInvolvement === 'none'`
- Prior-period delta
- Empty window returns zeros, not NaN
- Issue with multiple friction labels counted in each applicable bucket

Update `fixtures/ai-impact/rfe-data.json` with sample friction labels for demo mode.

### 3. New UI component

**File:** `modules/ai-impact/client/components/PipelineHealthRow.vue` (NEW)

- Props: `pipelineHealth` (object from API)
- Layout: 4-column grid matching `MetricsRow.vue` styling
- Section heading: **Pipeline Health** with subtitle: *Where the automation needs human help*
- Use amber/red tones for friction metrics (inverse of adoption green)

### 4. Wire into RFE Review

**Files:**
- `modules/ai-impact/client/views/RFEReviewView.vue` — pass `pipelineHealth` computed from `rfeData`
- `modules/ai-impact/client/components/PhaseContent.vue` — render `PipelineHealthRow` below `MetricsRow`
- Update header subtitle from "AI adoption metrics and RFE tracking" to mention pipeline health

---

## Phase 2: PM Breakdown + List Integration

### 1. PM breakdown table

**File:** `modules/ai-impact/client/components/PipelineHealthBreakdown.vue` (NEW)

- Collapsible section below Pipeline Health tiles
- Table grouped by `creatorDisplayName`
- Only include PMs with ≥1 AI-touched RFE in window (configurable minimum)

Server-side: add `buildPipelineHealthByCreator(issues, timeWindow)` to `metrics.js` and return in `pipelineHealth.byCreator`.

### 2. RFE list filters

**File:** `modules/ai-impact/client/components/RFEList.vue`

Add filter option (dropdown or pills):
- Needs attention
- Feasibility fail
- Feasibility unknown
- Rubric pass

Filter client-side against `rfe.labels.includes(...)`.

Optional: clicking a PM row in breakdown table sets creator search filter.

---

## Phase 3 (Optional): Trends + Topic Grouping

### Trend chart
- Add friction rate series to `TrendCharts.vue` or a sibling chart
- Document known limitation: bucketed by creation date, not label application date

### Topic grouping
Requires one of:
- Add `components` to RFE Jira fetch in `rfe-fetcher.js`, or
- Join to linked RHAISTRAT feature component, or
- Use assessment `antiPatterns` as proxy

Defer until PM/design confirms the grouping dimension.

---

## User Stories

### US-1: See pipeline friction at a glance

**As a PM lead, I want to see how often the rfe-creator pipeline requires human cleanup so I can gauge true automation efficiency.**

**Acceptance criteria:**
- Pipeline Health section visible on RFE Review below adoption metrics
- Shows needs-attention rate for the selected time window
- Shows change vs prior period

### US-2: Track feasibility blockers

**As a PM, I want to see feasibility fail and unknown rates so I can identify systemic pipeline quality issues.**

**Acceptance criteria:**
- Feasibility fail % and unknown % displayed
- Rates respect the same time window selector as adoption metrics

### US-3: Identify PMs needing support

**As an AI First lead, I want a breakdown of friction by PM so I can target training or prompt improvements.**

**Acceptance criteria:**
- Table shows per-PM friction counts and rates
- Sortable by highest needs-attention rate
- Does not expose data for PMs with zero AI-touched RFEs in window

### US-4: Drill into friction tickets

**As a PM, I want to filter the RFE list to tickets flagged needs-attention so I can review what went wrong.**

**Acceptance criteria:**
- List filter for each friction label type
- Filter composes with existing search and pass/fail filters

---

## Out of Scope

- New AI Impact nav tab (Pipeline Health lives inside RFE Review only)
- New Jira projects or fetch pipelines
- Engineering jira-autofix metrics (already in **Jira AutoFix** view)
- Feature Review / Test Plan friction labels (`strat-creator-needs-attention`, `test-plan-rubric-fail`) — future extension using same pattern
- Real-time pipeline run logs or session abandonment (not in Jira labels today)
- Per-label changelog timestamps (would require parsing `extractLabelDate` for friction labels)

---

## Documentation Updates

When implemented, update in the same PR:
- `docs/DATA-FORMATS.md` — document `pipelineHealth` shape on RFE API response if persisted
- `fixtures/ai-impact/rfe-data.json` — add friction label examples
- End-user guide (if created) — explain Pipeline Health section on RFE Review

---

## Open Questions

1. **Denominator:** AI-touched only vs all RFEs in window? (Plan assumes AI-touched only.)
2. **Topic dimension:** What should "topic area" mean for PMs — Jira component, feature link, or rubric anti-pattern?
3. **Alerting:** Should high needs-attention rate trigger an app-wide banner (see `docs/plans/field-completeness-alert.md`)? Deferred.
4. **Label ownership:** Should friction label strings move to `config.js` alongside `createdLabel` / `revisedLabel`?
