# Power BI Agent Orchestration — Architecture & Workflow

> **Last updated:** 2026-03-30
>
> This document describes every agent in the Power BI report factory, what
> each one does, what it produces, and how the agents interact through the
> orchestration pipeline.

---

## Table of Contents

1. [Agent Inventory](#agent-inventory-11-agents)
2. [Agent Deep Dives](#agent-deep-dives)
3. [Orchestration Pipeline (9 Steps)](#orchestration-pipeline-9-steps)
4. [Data Flow Between Agents](#data-flow-between-agents)
5. [Agent Interaction Matrix](#agent-interaction-matrix)
6. [Error Recovery Loops](#error-recovery-loops)
7. [Utility Agents (On-Demand)](#utility-agents-on-demand)
8. [Theme Enforcement](#theme-enforcement-every-agent)
9. [Standing Design Rules](#standing-design-rules)

---

## Agent Inventory (11 Agents)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AGENT ROSTER                                 │
├─────────────────────┬───────────────────────────────────────────────┤
│  PLANNING AGENTS    │  spec-writer        — Requirements → spec.md │
│                     │  data-profiler      — Data → Profile Report  │
│                     │  layout-strategist  — Spec → Layout Variants │
│                     │  ab-evaluator       — Variants → Winner      │
├─────────────────────┼───────────────────────────────────────────────┤
│  BUILD AGENTS       │  powerbi-engineer   — Builds PBIP projects   │
│                     │  semantic-model-analyzer — Data layer fixes   │
├─────────────────────┼───────────────────────────────────────────────┤
│  QUALITY AGENTS     │  report-tester      — Validates correctness  │
│                     │  designer           — Reviews visual quality  │
├─────────────────────┼───────────────────────────────────────────────┤
│  UTILITY AGENTS     │  kusto-expert       — KQL query building     │
│                     │  QandA              — Natural language queries│
├─────────────────────┼───────────────────────────────────────────────┤
│  COORDINATOR        │  report-orchestrator — Manages the pipeline  │
└─────────────────────┴───────────────────────────────────────────────┘
```

---

## Agent Deep Dives

### 1. report-orchestrator (Coordinator)

| Attribute | Detail |
|---|---|
| **Role** | Project manager and pipeline coordinator. Never builds or edits PBIP files directly. |
| **Category** | Coordinator |
| **Inputs** | User requirements (verbal or written) |
| **Outputs** | Orchestrated report delivery; progress updates; optional Fabric publication |
| **Delegates to** | Every other agent in the roster |
| **Invoked by** | The user (entry point for new report requests) |

The orchestrator owns the entire 9-step pipeline. It gathers requirements,
dispatches work to specialized agents, interprets their results, manages error
budgets (5 fix attempts, 3 design rounds), and presents the final deliverable.
It tracks pipeline state — current phase, attempt counts, latest test/design
results — and keeps the user informed at every transition.

**Key behaviors:**
- Asks focused questions one at a time during intake (never bundles)
- Auto-tests the first build without prompting; prompts for subsequent iterations
- Passes tester errors and designer suggestions verbatim to the engineer
- Never asks an agent to fix errors AND implement design changes simultaneously
- Updates `spec.md` first when requirements change mid-flight

---

### 2. spec-writer (Planning)

| Attribute | Detail |
|---|---|
| **Role** | Deep requirements analyst that transforms vague stakeholder input into a precise, structured specification. |
| **Category** | Planning |
| **Inputs** | User requirements summary from the orchestrator's intake phase |
| **Outputs** | `spec.md` — the single source of truth for all downstream agents |
| **Delegates to** | None (may interview the user for gaps) |
| **Invoked by** | report-orchestrator (Step 2) |

The spec-writer interviews the user to fill any gaps the orchestrator's intake
didn't cover. It deep-dives into KPI definitions, maps them to data source
columns, structures page layouts, and codifies the premium dark blue theme. The
resulting `spec.md` is the contract — every downstream agent reads it.

**What spec.md contains:**
- Audience & purpose
- Data source connection details
- KPI definitions with formulas and data bindings
- Page-by-page layout descriptions
- Filter/slicer specifications
- Theme colors and typography
- Acceptance criteria

---

### 3. data-profiler (Planning)

| Attribute | Detail |
|---|---|
| **Role** | Profiles actual data sources before design begins, discovering what the data really looks like. |
| **Category** | Planning |
| **Inputs** | Data source connection details from `spec.md` |
| **Outputs** | Data Profile Report (column stats, cardinality, distributions, null rates, date ranges, KPI recommendations) |
| **Delegates to** | None (connects directly to data sources) |
| **Invoked by** | report-orchestrator (Step 3) |

**Supported data sources:**
- Kusto / Azure Data Explorer (KQL)
- SharePoint lists
- Excel files
- SQL databases
- CSV files

**When to run:**
- **Always** for Kusto/ADX and SQL (complex data, schema discovery is high-value)
- **Recommended** for SharePoint and Excel (data quality often varies)
- **Skip** only if the user explicitly says "I already know my data"

**What the profile feeds downstream:**
- The layout-strategist uses cardinality data to pick good slicer dimensions
- The layout-strategist uses distribution data to choose appropriate chart types
- The powerbi-engineer uses type information and null rates for robust modeling

---

### 4. layout-strategist (Planning)

| Attribute | Detail |
|---|---|
| **Role** | Generates 2–3 competing layout blueprints using distinct design archetypes, enabling data-driven layout selection. |
| **Category** | Planning |
| **Inputs** | `spec.md` + Data Profile Report (if available) |
| **Outputs** | 2–3 layout blueprints with pixel-precise positions, visual types, data bindings, spacing, and rationale |
| **Delegates to** | None |
| **Invoked by** | report-orchestrator (Step 4) |

**Design archetypes it draws from:**
| Archetype | Character |
|---|---|
| **Glanceable** | KPI-forward with large cards, minimal charts, scannable in <5 seconds |
| **Narrative** | Story-driven flow with charts that guide the reader through a logical progression |
| **Command Center** | Data-dense with matrices, tables, and comprehensive drill-down; for power users |

Each blueprint includes:
- Page-by-page visual placement (x, y, width, height in pixels)
- Visual type for each position (card, matrix, bar chart, slicer, etc.)
- Data field bindings
- Spacing and alignment rationale
- Pros/cons for the target audience

---

### 5. ab-evaluator (Planning)

| Attribute | Detail |
|---|---|
| **Role** | Compares 2–3 layout blueprints side-by-side using a structured scoring framework and recommends the best one. |
| **Category** | Planning |
| **Inputs** | Layout blueprints from layout-strategist; `spec.md` for context; optional PBIP files or screenshots for post-build comparison |
| **Outputs** | Structured scorecard with per-dimension scores + winner recommendation (may suggest a hybrid) |
| **Delegates to** | None |
| **Invoked by** | report-orchestrator (Step 5) |

**Scoring framework (8 dimensions):**

| Dimension | Weight | What it measures |
|---|---|---|
| First Impression | 2× | Can a VP get the story in 5 seconds? |
| Visual Hierarchy | 2× | Is there a clear reading order? |
| Data Storytelling | 2× | Do the visuals tell a coherent narrative? |
| Info Density | 1× | Is space used efficiently without clutter? |
| Theme Adherence | 1× | Does it follow the premium dark blue palette? |
| Readability | 1× | Can text be read on a projector at 10 feet? |
| Interaction Design | 1× | Are slicers and drill-downs intuitive? |
| Scalability | 1× | Will it handle 2× the data gracefully? |

**Max possible: 55 points**

**Possible outcomes:**
| Result | Orchestrator action |
|---|---|
| Clear winner | Proceed to BUILD with the winning blueprint |
| Hybrid recommended | Create combined blueprint, confirm with user, then BUILD |
| Tied / user preference | Present comparison to user, let them choose |

---

### 6. powerbi-engineer (Build)

| Attribute | Detail |
|---|---|
| **Role** | The builder. Creates production-ready PBIP projects and semantic models in the PBIR enhanced format. |
| **Category** | Build |
| **Inputs** | `spec.md`, winning layout blueprint, fix instructions from tester, design suggestions from designer |
| **Outputs** | Complete PBIP project on disk (`.pbip`, `.pbir`, TMDL, `visual.json` files, DAX measures, theme, etc.) |
| **Delegates to** | None (uses file system tools to create/edit files) |
| **Invoked by** | report-orchestrator (Step 6, fix loops, design loops) |

**What it builds:**
- Full PBIP folder structure (`.pbip`, `.pbir`, `definition/` trees)
- TMDL semantic model (tables, columns, measures, relationships)
- `visual.json` for every visual on every page
- DAX measures for calculated KPIs
- Dark blue theme configuration
- Data source connections (Kusto, SharePoint, Excel, SQL)
- Bookmarks, drillthrough pages, conditional formatting
- Multi-table relationships

**Three modes the orchestrator uses it in:**

| Mode | When | Prompt pattern |
|---|---|---|
| **Initial build** | Step 6, first time | "Build a complete PBIP project at `<path>`. Read spec.md. Use this layout blueprint..." |
| **Fix build** | After tester FAIL | "Fix the following errors in the PBIP at `<path>`. The report-tester found..." |
| **Design build** | After designer feedback | "Apply these design improvements to the PBIP at `<path>`. The designer said..." |

**Also handles:**
- Publishing to Microsoft Fabric workspaces
- Refreshing data connections post-publish

---

### 7. semantic-model-analyzer (Build)

| Attribute | Detail |
|---|---|
| **Role** | Deep data-layer analyst that detects schema mismatches, type conflicts, and missing transformations that cause import errors or incorrect DAX results. |
| **Category** | Build (optional specialist) |
| **Inputs** | TMDL partition files, embedded KQL/SQL/M queries |
| **Outputs** | M transformation layers (`Table.TransformColumnTypes`, `Table.SelectColumns`, null handling, type coercion) + data-layer analysis report |
| **Delegates to** | None |
| **Invoked by** | report-orchestrator (optional, after tester passes or when data-layer issues suspected) |

**What it detects:**
| Issue | Description |
|---|---|
| **Union null-risk** | `\| union` combining different data shapes → DAX overcounting |
| **Pivot schema drift** | `evaluate pivot()` columns that depend on data values → may vanish on refresh |
| **Type mismatches** | KQL `dynamic` → TMDL `string`, `int` vs `long` conflicts |
| **Missing M transforms** | No `Table.SelectColumns`, no type coercion, no null handling |

**When to invoke:**
- Complex KQL with multi-join, cross-cluster, or `| union` patterns
- Tester reports warnings about M partitions or KQL queries
- User explicitly requests a data-layer review

**When to skip:**
- Simple CSV/Excel imports
- Projects already refreshed successfully in PBI Desktop

---

### 8. report-tester (Quality)

| Attribute | Detail |
|---|---|
| **Role** | QA validator that checks PBIP projects for structural and runtime correctness. |
| **Category** | Quality |
| **Inputs** | PBIP project on disk |
| **Outputs** | Structured pass/fail report with actionable error details and "Errors for Engineer" section |
| **Delegates to** | None (may launch PBI Desktop for Phase 2) |
| **Invoked by** | report-orchestrator (Step 7, fix loop iterations) |

**Two-phase validation:**

| Phase | What it checks | Requires PBI Desktop? |
|---|---|---|
| **Phase 1** — Schema & structural | JSON schema, TMDL syntax, page/visual cross-references, 20+ known gotchas, folder structure | No |
| **Phase 2** — PBI Desktop integration | Opens the `.pbip` file, reads trace logs, classifies errors by severity | Yes |

**Result categories:**
| Result | Meaning | Orchestrator action |
|---|---|---|
| **PASS** | No errors | Proceed to design review |
| **PASS WITH WARNINGS** | Minor issues, non-blocking | Proceed to design, note warnings |
| **FAIL** | Blocking errors found | Loop back to BUILD with error details |

**Error budget:** Maximum 5 fix cycles (BUILD → TEST). After 5 failures, the
orchestrator stops and escalates to the user.

---

### 9. designer (Quality)

| Attribute | Detail |
|---|---|
| **Role** | Executive dashboard design critic. Reviews visual hierarchy, layout balance, color, typography, data storytelling, and projector readability. |
| **Category** | Quality |
| **Inputs** | PBIP project on disk + `spec.md` for context; optional screenshots |
| **Outputs** | Prioritized design suggestions (Priority 1, 2, 3) with specific file/property references + letter grade (A–F) |
| **Delegates to** | None (read-only — never modifies files) |
| **Invoked by** | report-orchestrator (Step 8, design loop iterations) |

**What it evaluates:**
- Visual hierarchy and reading order
- Color consistency against the theme
- Typography (size, weight, contrast)
- Data storytelling (do visuals tell a coherent narrative?)
- Projector/large-screen readability
- Layout balance and whitespace
- Executive "5-second scan" effectiveness

**Grade interpretation:**
| Grade | Orchestrator action |
|---|---|
| **A** | Done — proceed to FINAL |
| **B** with ≤2 suggestions | Good enough — proceed to FINAL, share suggestions with user |
| **B** with 3+ suggestions | Loop back to BUILD with Priority 1 + 2 suggestions |
| **C / D / F** | Loop back to BUILD with Priority 1 + 2 suggestions |

**Design budget:** Maximum 3 polish cycles (BUILD → TEST → DESIGN). After 3
rounds, the orchestrator presents the report as-is.

---

### 10. kusto-expert (Utility)

| Attribute | Detail |
|---|---|
| **Role** | Interactive KQL query builder and analyzer for Azure Data Explorer (Kusto/ADX). |
| **Category** | Utility (on-demand) |
| **Inputs** | User questions about Kusto data; cluster/database connection details |
| **Outputs** | Tested KQL queries, performance insights, data quality warnings, interesting findings |
| **Delegates to** | None |
| **Invoked by** | User directly, or report-orchestrator when KQL expertise is needed |

**Capabilities:**
- Connects to Kusto clusters and explores schemas
- Generates optimized KQL queries through interactive conversation
- Runs and tests queries, returning results
- Identifies performance bottlenecks and suggests optimizations
- Surfaces data quality issues and interesting patterns

**Not part of the main pipeline** — available anytime the user needs help with
KQL, whether for report data sources or ad-hoc analysis.

---

### 11. QandA (Utility)

| Attribute | Detail |
|---|---|
| **Role** | Natural-language Q&A interface for published Power BI reports and semantic models in Microsoft Fabric. |
| **Category** | Utility (on-demand) |
| **Inputs** | User questions in plain English about report data |
| **Outputs** | Answers in natural language, backed by generated and executed DAX queries |
| **Delegates to** | None |
| **Invoked by** | User directly |

**How it works:**
1. Discovers available reports and semantic models in Fabric
2. Lets the user pick which model to query
3. Fetches the model schema
4. Generates DAX from the user's natural-language question
5. Executes the DAX and explains the results

**Not part of the main pipeline** — used post-build when a user wants to
interrogate a published report's data without writing DAX themselves.

---

## Orchestration Pipeline (9 Steps)

```
    YOU (User)
     │
     │  "I need a report showing service health for my VP"
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   REPORT-ORCHESTRATOR                                │
│            (Coordinates everything — never builds)                   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 1: INTAKE                                               │   │
│  │                                                              │   │
│  │  Orchestrator asks focused questions one at a time:          │   │
│  │  Q1  Who is this for? (VP / Manager / Analyst)               │   │
│  │  Q2  Where's the data? (Kusto / SharePoint / Excel / SQL)    │   │
│  │  Q3  What KPIs matter? (freeform)                            │   │
│  │  Q4  How many pages?                                         │   │
│  │  Q5  Page height? (720 / 1080 / 1320 / custom)              │   │
│  │  Q6  Visual types? (cards / charts / tables / mix)           │   │
│  │  Q7  Filters & slicers?                                      │   │
│  │  Q8  Color theme?                                            │   │
│  │  Q9  Section headers & dividers?                             │   │
│  │  Q10 Output location on disk?                                │   │
│  │  Q11 Publish to Fabric?                                      │   │
│  │                                                              │   │
│  │  Output: Confirmed requirements summary                      │   │
│  └──────────────────────┬───────────────────────────────────────┘   │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 2: SPEC                                                 │   │
│  │                                           ┌───────────────┐  │   │
│  │  Orchestrator invokes ──────────────────▶ │  SPEC-WRITER  │  │   │
│  │                                           │               │  │   │
│  │  The spec-writer:                         │  • Interviews  │  │   │
│  │  • Deep-dives into KPI definitions        │    for gaps   │  │   │
│  │  • Maps KPIs to data source columns       │  • Defines    │  │   │
│  │  • Structures page layouts                │    every KPI  │  │   │
│  │  • Sets premium dark blue theme           │  • Writes     │  │   │
│  │  • Produces spec.md                       │    spec.md    │  │   │
│  │                                           └───────┬───────┘  │   │
│  │  Output: spec.md (single source of truth)         │          │   │
│  └──────────────────────┬────────────────────────────┘          │   │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 3: PROFILE (Optional but Recommended)                   │   │
│  │                                           ┌───────────────┐  │   │
│  │  Orchestrator invokes ──────────────────▶ │ DATA-PROFILER │  │   │
│  │                                           │               │  │   │
│  │  The profiler connects to actual data:    │  • Connects   │  │   │
│  │  • Runs KQL/SQL queries against source    │    to source  │  │   │
│  │  • Discovers column stats, nulls, types   │  • Profiles   │  │   │
│  │  • Finds cardinality (good slicer dims)   │    every col  │  │   │
│  │  • Detects outliers, trends, gaps         │  • Recommends │  │   │
│  │  • Recommends KPIs from actual data       │    KPIs &     │  │   │
│  │                                           │    visuals    │  │   │
│  │  Output: Data Profile Report              └───────┬───────┘  │   │
│  │  (feeds the layout-strategist)                    │          │   │
│  └──────────────────────┬────────────────────────────┘          │   │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 4: LAYOUT VARIANTS                                      │   │
│  │                                           ┌────────────────┐ │   │
│  │  Orchestrator invokes ──────────────────▶ │LAYOUT-         │ │   │
│  │                                           │STRATEGIST      │ │   │
│  │  Reads spec.md + data profile, then:      │                │ │   │
│  │                                           │ Generates 2-3  │ │   │
│  │  ┌─────────────┐  ┌─────────────┐        │ blueprints     │ │   │
│  │  │ Variant A   │  │ Variant B   │        │ from different │ │   │
│  │  │"Glanceable" │  │"Narrative"  │ ◀──────│ archetypes     │ │   │
│  │  │ KPI-forward │  │ Story-forward│        └────────────────┘ │   │
│  │  │ Big cards   │  │ Charts+flow │                            │   │
│  │  └──────┬──────┘  └──────┬──────┘  ┌─────────────┐          │   │
│  │         │                │         │ Variant C   │          │   │
│  │         │                │         │"Cmd Center" │          │   │
│  │         │                │         │ Data-dense  │          │   │
│  │         │                │         └──────┬──────┘          │   │
│  │         ▼                ▼                ▼                  │   │
│  │  Each blueprint has pixel-precise positions,                 │   │
│  │  visual types, data bindings, and rationale                  │   │
│  └──────────────────────┬───────────────────────────────────────┘   │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 5: A/B EVALUATE                                         │   │
│  │                                           ┌───────────────┐  │   │
│  │  Orchestrator invokes ──────────────────▶ │ AB-EVALUATOR  │  │   │
│  │                                           │               │  │   │
│  │  Scores each variant on 8 dimensions:     │  Scoring:     │  │   │
│  │                                           │               │  │   │
│  │   Dimension          Weight               │  • Analyzes   │  │   │
│  │   ─────────────────  ──────               │    each       │  │   │
│  │   First Impression    2×                  │    variant    │  │   │
│  │   Visual Hierarchy    2×                  │  • Scores     │  │   │
│  │   Data Storytelling   2×                  │    1-5 on     │  │   │
│  │   Info Density        1×                  │    each dim   │  │   │
│  │   Theme Adherence     1×                  │  • Picks      │  │   │
│  │   Readability         1×                  │    winner     │  │   │
│  │   Interaction Design  1×                  │  • May suggest│  │   │
│  │   Scalability         1×                  │    hybrid     │  │   │
│  │                       ──                  └───────┬───────┘  │   │
│  │   Max possible: 55 points                         │          │   │
│  │                                                   │          │   │
│  │  Output: 🏆 Winner + scorecard + improvements     │          │   │
│  └──────────────────────┬────────────────────────────┘          │   │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 6: BUILD                                                │   │
│  │                                           ┌───────────────┐  │   │
│  │  Orchestrator invokes ──────────────────▶ │  POWERBI-     │  │   │
│  │  with winning blueprint + spec.md         │  ENGINEER     │  │   │
│  │                                           │               │  │   │
│  │  The engineer:                            │  • Creates    │  │   │
│  │  • Creates full PBIP folder structure     │    .pbip      │  │   │
│  │  • Writes TMDL semantic model             │  • Writes     │  │   │
│  │  • Builds visual.json for each visual     │    TMDL       │  │   │
│  │  • Applies dark blue theme everywhere     │  • Builds     │  │   │
│  │  • Wires up data connections              │    visuals    │  │   │
│  │  • Creates DAX measures                   │  • Applies    │  │   │
│  │                                           │    theme      │  │   │
│  │  Output: Complete PBIP project on disk    └───────┬───────┘  │   │
│  └──────────────────────┬────────────────────────────┘          │   │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 7: TEST                            ┌───────────────┐    │   │
│  │                                         │ REPORT-TESTER │    │   │
│  │  Orchestrator invokes ────────────────▶ │               │    │   │
│  │                                         │ Phase 1:      │    │   │
│  │  Two-phase validation:                  │  • JSON schema│    │   │
│  │                                         │  • TMDL syntax│    │   │
│  │  Phase 1: Schema & structural checks    │  • Page refs  │    │   │
│  │  (fast, no PBI Desktop needed)          │  • 20 gotchas │    │   │
│  │                                         │               │    │   │
│  │  Phase 2: PBI Desktop integration       │ Phase 2:      │    │   │
│  │  (launches PBI, checks trace logs)      │  • Opens .pbip│    │   │
│  │                                         │  • Reads logs │    │   │
│  │                                         │  • Classifies │    │   │
│  │  ┌─────────┐                            │    errors     │    │   │
│  │  │  FAIL   │◀── errors found ───────────└───────┬───────┘    │   │
│  │  │         │                                    │            │   │
│  │  │ Loop ◀──┘                                    │            │   │
│  │  │ back to    ◀─── max 5 fix attempts ──────────┘            │   │
│  │  │ Step 6                                                    │   │
│  │  │ (BUILD)    PASS ─────────────────────────────────▶        │   │
│  │  └─────────┘                                                 │   │
│  └──────────────────────┬───────────────────────────────────────┘   │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 8: DESIGN REVIEW                   ┌───────────────┐    │   │
│  │                                         │   DESIGNER    │    │   │
│  │  Orchestrator invokes ────────────────▶ │               │    │   │
│  │                                         │  • Layout     │    │   │
│  │  The designer reviews:                  │    analysis   │    │   │
│  │  • Visual hierarchy                     │  • Color      │    │   │
│  │  • Color consistency                    │    audit      │    │   │
│  │  • Typography                           │  • Typography │    │   │
│  │  • Data storytelling                    │    audit      │    │   │
│  │  • Projector readability                │  • Screenshot │    │   │
│  │                                         │    review     │    │   │
│  │  ┌─────────┐                            │  • Grades     │    │   │
│  │  │ Grade   │◀── C/D/F or B with ────────│    A-F        │    │   │
│  │  │ < B     │    3+ suggestions          └───────┬───────┘    │   │
│  │  │         │                                    │            │   │
│  │  │ Loop ◀──┘                                    │            │   │
│  │  │ back to    ◀─── max 3 design rounds ─────────┘            │   │
│  │  │ Step 6                                                    │   │
│  │  │ (BUILD)    Grade A/B ────────────────────────────▶        │   │
│  │  └─────────┘                                                 │   │
│  └──────────────────────┬───────────────────────────────────────┘   │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 9: FINAL                                                │   │
│  │                                                              │   │
│  │  ✅ Report Complete!                                         │   │
│  │                                                              │   │
│  │  📁 Location: C:\...\MyReport                                │   │
│  │  🎨 Theme: Premium Dark Blue (#2A324B)                       │   │
│  │  📐 Design Grade: A                                          │   │
│  │  🏆 Layout: "Narrative" (won A/B with 49/55)                 │   │
│  │  ✓  Validation: PASS                                         │   │
│  │                                                              │   │
│  │  → Optionally publish to Microsoft Fabric workspace          │   │
│  │    (via powerbi-engineer)                                    │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow Between Agents

This diagram shows **what artifacts flow between agents** and in what direction.

```
                              spec.md
                          (source of truth)
                                │
                 ┌──────────────┼──────────────────────────┐
                 │              │                           │
                 ▼              ▼                           ▼
          ┌────────────┐ ┌────────────┐            ┌──────────────┐
          │  data-     │ │  layout-   │            │  powerbi-    │
          │  profiler  │ │  strategist│            │  engineer    │
          └─────┬──────┘ └─────┬──────┘            └──────┬───────┘
                │              │                          │
        Data Profile     Layout Blueprints          PBIP Project
           Report         (2-3 variants)            (on disk)
                │              │                          │
                └──────┬───────┘                   ┌──────┴───────┐
                       ▼                           ▼              ▼
                ┌────────────┐              ┌──────────┐   ┌──────────┐
                │    ab-     │              │  report- │   │ designer │
                │  evaluator │              │  tester  │   │          │
                └─────┬──────┘              └──────┬───┘   └────┬─────┘
                      │                           │             │
               Winner + Scorecard          Validation      Design Grade
                      │                    Report          + Suggestions
                      │                           │             │
                      └───────────┬───────────────┴─────────────┘
                                  ▼
                          report-orchestrator
                          (routes everything)
```

### Artifact Registry

Every piece of data that moves between agents:

| Artifact | Produced by | Consumed by | Format |
|---|---|---|---|
| `spec.md` | spec-writer | all pipeline agents | Markdown |
| Data Profile Report | data-profiler | layout-strategist, orchestrator | Structured text |
| Layout Blueprints (×2–3) | layout-strategist | ab-evaluator, orchestrator | Structured text with pixel specs |
| Winner Scorecard | ab-evaluator | orchestrator → powerbi-engineer | Structured text |
| PBIP Project (on disk) | powerbi-engineer | report-tester, designer | PBIP/PBIR file tree |
| Validation Report | report-tester | orchestrator → powerbi-engineer (on fail) | Structured pass/fail |
| Design Review | designer | orchestrator → powerbi-engineer (on low grade) | Prioritized suggestions |
| M Transformation Code | semantic-model-analyzer | powerbi-engineer | M/Power Query code |
| KQL Queries | kusto-expert | user, powerbi-engineer | KQL text |
| DAX Results | QandA | user | Natural language + tables |

---

## Agent Interaction Matrix

Shows which agents **directly communicate** (via the orchestrator routing
messages between them). ● = primary flow, ○ = optional/conditional flow.

```
                      RECEIVES OUTPUT FROM →
                  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
                  │ ORC │ SPW │ DPR │ LST │ ABE │ ENG │ SMA │ TST │ DES │ KUS │ Q&A │
  ┌───────────────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┤
  │ orchestrator  │  -  │  ●  │  ●  │  ●  │  ●  │  ●  │  ○  │  ●  │  ●  │  ○  │     │
  │ spec-writer   │  ●  │  -  │     │     │     │     │     │     │     │     │     │
  │ data-profiler │  ●  │     │  -  │     │     │     │     │     │     │     │     │
  │ layout-strat  │  ●  │  ●  │  ●  │  -  │     │     │     │     │     │     │     │
  │ ab-evaluator  │  ●  │     │     │  ●  │  -  │     │     │     │     │     │     │
  │ pbi-engineer  │  ●  │  ●  │     │  ●  │  ●  │  -  │  ○  │  ●  │  ●  │     │     │
  │ sem-analyzer  │  ○  │     │     │     │     │  ○  │  -  │     │     │     │     │
  │ report-tester │  ●  │     │     │     │     │  ●  │     │  -  │     │     │     │
  │ designer      │  ●  │  ●  │     │     │     │  ●  │     │     │  -  │     │     │
  │ kusto-expert  │  ○  │     │     │     │     │     │     │     │     │  -  │     │
  │ QandA         │     │     │     │     │     │     │     │     │     │     │  -  │
  └───────────────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘

  Legend: ORC = orchestrator, SPW = spec-writer, DPR = data-profiler,
          LST = layout-strategist, ABE = ab-evaluator, ENG = powerbi-engineer,
          SMA = semantic-model-analyzer, TST = report-tester, DES = designer,
          KUS = kusto-expert, Q&A = QandA
```

**Key relationships explained:**

| Relationship | How it works |
|---|---|
| **orchestrator → spec-writer** | Passes confirmed intake summary; receives spec.md |
| **orchestrator → data-profiler** | Passes data source details from spec; receives profile report |
| **orchestrator → layout-strategist** | Passes spec.md + profile findings; receives 2–3 blueprints |
| **orchestrator → ab-evaluator** | Passes layout blueprints; receives scorecard + winner |
| **orchestrator → powerbi-engineer** | Passes spec + blueprint (build), error list (fix), or design suggestions (polish); receives completed PBIP |
| **orchestrator → report-tester** | Points tester at PBIP on disk; receives pass/fail report |
| **orchestrator → designer** | Points designer at PBIP + spec; receives grade + suggestions |
| **report-tester → powerbi-engineer** | Tester errors are routed through the orchestrator to the engineer for fixes |
| **designer → powerbi-engineer** | Design suggestions are routed through the orchestrator to the engineer for implementation |
| **semantic-model-analyzer → powerbi-engineer** | M transformation code is routed through the orchestrator to the engineer for integration |
| **spec-writer → all agents** | spec.md is the contract that all agents read |

---

## Error Recovery Loops

### Fix Loop (BUILD ↔ TEST)

```
  BUILD ──────▶ TEST
    ▲              │
    │         FAIL │
    │        (max  │
    │       5 fix  │
    │      rounds) │
    └──────────────┘
         │
    After 5 failures:
    → Stop and escalate to user
    → User chooses: retry 3 more, skip, or fix manually
```

### Design Loop (BUILD ↔ TEST ↔ DESIGN)

```
  BUILD ──▶ TEST ──▶ DESIGN
    ▲                   │
    │         C/D/F or  │
    │         B with    │
    │         3+ issues │
    │        (max 3     │
    │       design      │
    │       rounds)     │
    └───────────────────┘
         │
    After 3 rounds:
    → Present report as-is with latest grade
    → User chooses: specific changes, publish, or done
```

### Combined Flow with Budgets

```
                    ┌─────────────────────┐
                    │   BUILD (Step 6)    │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   TEST  (Step 7)    │
                    └──────────┬──────────┘
                          ┌────┴────┐
                        FAIL       PASS
                          │          │
                   ┌──────▼──────┐   │
                   │ Fix attempt │   │
                   │   N/5       │   │
                   └──────┬──────┘   │
                     N<5? │          │
                    ┌─────┴─────┐    │
                   YES         NO    │
                    │           │    │
                    ▼           ▼    ▼
                  BUILD    ESCALATE  │
                               TO   │
                              USER  │
                                    │
                    ┌───────────────▼──────┐
                    │  DESIGN  (Step 8)    │
                    └──────────┬───────────┘
                          ┌────┴────┐
                      Grade A/B    Grade C-F
                      (≤2 sug.)    or B(3+)
                          │          │
                          ▼     ┌────▼──────┐
                        DONE    │ Design    │
                                │ round M/3 │
                                └─────┬─────┘
                                 M<3? │
                               ┌──────┴──────┐
                              YES            NO
                               │              │
                               ▼              ▼
                             BUILD         PRESENT
                                           AS-IS
```

---

## Utility Agents (On-Demand)

These agents are **not part of the main 9-step pipeline** but can be invoked
at any time by the user or by the orchestrator when specialized expertise is
needed.

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  ┌──────────────────┐                                               │
│  │   KUSTO-EXPERT   │  When:                                        │
│  │                  │  • User needs help writing KQL queries         │
│  │  Interactive KQL │  • Exploring a Kusto cluster schema            │
│  │  query builder   │  • Optimizing slow queries                     │
│  │                  │  • Ad-hoc data exploration                     │
│  └──────────────────┘                                               │
│                                                                     │
│  ┌──────────────────┐                                               │
│  │      Q&A         │  When:                                        │
│  │                  │  • User wants to ask questions about a         │
│  │  Natural-language│    published report in plain English           │
│  │  report querying │  • "What was the top region last quarter?"     │
│  │                  │  • Generates and executes DAX behind the scenes│
│  └──────────────────┘                                               │
│                                                                     │
│  ┌──────────────────┐                                               │
│  │ SEMANTIC-MODEL-  │  When:                                        │
│  │ ANALYZER         │  • Complex KQL unions need schema validation   │
│  │                  │  • Type mismatches between KQL and TMDL        │
│  │  Data-layer      │  • Tester reports M partition warnings         │
│  │  deep analysis   │  • `evaluate pivot()` columns may drift       │
│  │                  │  • User requests a data-layer review           │
│  └──────────────────┘                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Theme Enforcement (Every Agent)

The premium dark blue theme is **non-negotiable** and is enforced by the
orchestrator in every prompt to the powerbi-engineer and designer. The only
exception is when the user explicitly requests a different theme during intake.

```
  Premium Dark Blue Palette
  ─────────────────────────
  #2A324B  ██████  Page background
  #353F5A  ██████  Card surfaces
  #4A90D9  ██████  Accent / column headers
  #FFFFFF  ██████  Primary text
  #B0B8C8  ██████  Secondary text
  #4CAF50  ██████  Success / green
  #E8A838  ██████  Warning / amber
  #FF4444  ██████  Critical / red
```

---

## Standing Design Rules

These rules are **always included** in prompts to the powerbi-engineer and
designer. They represent learned preferences that produce the best results:

| Rule | Detail |
|---|---|
| **Card titles** | NEVER shown unless user explicitly requests them |
| **Slicer headers** | NEVER shown unless user explicitly requests them |
| **Filters/slicers** | Always at the TOP of the page |
| **No "Filters" label** | Slicers sit directly below title/subtitle with no section header or divider |
| **Table/matrix row backgrounds** | Match the report background color (no alternating rows) |
| **Table/matrix column headers** | Blue `#4A90D9`, white text, Grow to Fit enabled |
| **Table/matrix grid** | Horizontal lines only (`#353F5A`), no vertical lines |
| **Table/matrix totals** | Off by default |
| **Section headers** | Go ABOVE their divider lines, not below |
| **Page heights** | Tight — approximately 30px below the lowest visual; pages can differ in height |

---

## Reference Projects

| Project | Location | Use for |
|---|---|---|
| **ADOWorkItemsReport** | `C:\Users\cameronbayer\Documents\ADOWorkItemsReport` | Canonical `.platform` format, `pages.json` schema, `report.json` schema. Opens cleanly in PBI Desktop. |
| **S360Dashboard** | `C:\Users\cameronbayer\OneDrive - Microsoft\Desktop\Projects\S360Dashboard` | 2-page dark-theme report with live Kusto data, complex KQL unions, matrix with 6-level drill-down, `visualContainer/2.8.0` schema. |

When the tester or engineer encounters structural errors, compare against
these known-good projects to identify format discrepancies.
