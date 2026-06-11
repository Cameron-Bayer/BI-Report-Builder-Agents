# Report Factory — Architecture & Workflow

> **Last updated:** 2025-07-14
>
> This document describes every agent in the report factory, what each one
> does, what it produces, and how the agents interact through the
> orchestration pipeline. The factory supports **multiple reporting
> frameworks** (Power BI, Grafana, HTML, and future additions) through a
> shared contract that framework-specific agents implement.

---

## Table of Contents

1. [Framework Contract](#framework-contract)
2. [Agent Inventory](#agent-inventory)
3. [Agent Deep Dives](#agent-deep-dives)
4. [Orchestration Pipeline (10 Steps)](#orchestration-pipeline-10-steps)
5. [Data Flow Between Agents](#data-flow-between-agents)
6. [Agent Interaction Matrix](#agent-interaction-matrix)
7. [Error Recovery Loops](#error-recovery-loops)
8. [Utility Agents (On-Demand)](#utility-agents-on-demand)
9. [Theme Enforcement](#theme-enforcement)
10. [Standing Design Rules](#standing-design-rules)
11. [Reference Projects](#reference-projects)

---

## Framework Contract

All agents in the report factory share a common vocabulary defined by this
contract. Framework-specific agents implement the contract for their framework.

### Supported Frameworks

| Framework | Engineer Agent | Tester Agent | Maturity | Delivery Target |
|---|---|---|---|---|
| **Power BI** | powerbi-engineer | powerbi-tester | Production | PBIP project → Fabric workspace |
| **Grafana** | grafana-engineer | grafana-tester | Experimental | Grafana dashboard JSON → Grafana instance |
| **HTML** | html-engineer | html-tester | Experimental | Static HTML/CSS/JS → file system or web server |

### Framework Capability Matrix

| Capability | Power BI | Grafana | HTML |
|---|---|---|---|
| Semantic model / data layer | ✅ (TMDL) | ✅ (datasources) | ❌ (data embedded or fetched) |
| Live data connection | ✅ | ✅ | ⚠️ (via JS fetch) |
| Layout blueprints | ✅ | ✅ | ✅ |
| A/B layout evaluation | ✅ | ✅ | ✅ |
| Structural validation | ✅ (Phase 1 + Phase 2) | ✅ (JSON schema) | ✅ (HTML lint + accessibility) |
| Desktop/runtime test | ✅ (PBI Desktop) | ⚠️ (render preview) | ✅ (browser render) |
| Design review | ✅ | ✅ | ✅ |
| Screenshot capture | ✅ | ⚠️ | ✅ |
| Publish to cloud | ✅ (Fabric) | ✅ (Grafana API) | ⚠️ (manual deploy) |
| Drillthrough / navigation | ✅ | ✅ (links) | ✅ (anchor links) |
| Interactive filters | ✅ (slicers) | ✅ (variables) | ⚠️ (JS-based) |
| Data refresh | ✅ (scheduled) | ✅ (datasource) | ❌ (static) |

### Normalized Visual Types

These are the cross-framework visual categories. Each framework maps them to
native implementations.

| Normalized Type | Power BI | Grafana | HTML |
|---|---|---|---|
| `kpi-card` | card visual | Stat panel | `<div>` with styled number |
| `bar-chart` | barChart / columnChart | Bar chart panel | Chart.js / D3 bar |
| `line-chart` | lineChart | Time series panel | Chart.js / D3 line |
| `table` | tableEx | Table panel | `<table>` element |
| `matrix` | pivotTable | Table with grouping | Nested `<table>` or tree grid |
| `donut-chart` | donutChart | Pie chart panel | Chart.js doughnut |
| `gauge` | gauge | Gauge panel | SVG gauge |
| `text-block` | textbox | Text panel | `<div>` / `<p>` |
| `filter` | slicer | Dashboard variable | `<select>` / custom filter UI |
| `divider` | shape (line) | — (panel spacing) | `<hr>` |
| `scatter` | scatterChart | XY Chart panel | Chart.js scatter |
| `map` | map | Geomap panel | Leaflet / Mapbox |

### Theme Token Schema

Themes are defined as a set of tokens that each framework interprets. This
replaces hardcoded hex values across agents.

```yaml
theme:
  name: "Premium Dark Blue"           # Human-readable name
  type: dark | light                   # Base mode
  tokens:
    page-background: "#2A324B"         # Page/dashboard background
    surface: "#353F5A"                 # Card/panel surfaces
    accent: "#4A90D9"                  # Highlights, headers, active elements
    text-primary: "#FFFFFF"            # Main body text
    text-secondary: "#B0B8C8"         # Subtitles, labels
    border: "#353F5A"                  # Grid lines, subtle borders
    success: "#4CAF50"                # Positive indicators
    warning: "#E8A838"                # Caution indicators
    critical: "#FF4444"               # Alert indicators
    divider: "#4A90D9"                # Section separators
  typography:
    family: "Segoe UI, system-ui, sans-serif"
    title-size: "20-24pt"
    subtitle-size: "10-12pt"
    body-size: "9-12pt"
    kpi-value-size: "24-28pt"
    kpi-label-size: "10pt"
  spacing:
    gutter: 16                        # px between visuals
    margin: 16                        # px page margin
    canvas-width: 1280                # px default width
    canvas-height: 720                # px default height
```

### Default Themes Per Framework

| Framework | Default Theme | Notes |
|---|---|---|
| Power BI | Premium Dark Blue | Executive dashboards expect dark, polished look |
| Grafana | Grafana Dark | Matches Grafana's native dark mode |
| HTML | Clean Light | Light backgrounds are more natural for web reports |

### Normalized Artifacts

These are the intermediate artifacts that flow between agents, regardless of
framework:

| Artifact | Description | Framework-Specific Extension |
|---|---|---|
| `spec.md` | Requirements specification | Includes `framework:` field |
| Layout Blueprint | Visual placements, types, sizes | Visual types mapped to normalized types |
| Validation Report | Pass/fail with actionable errors | Error format varies by framework |
| Design Review | Grade + prioritized suggestions | File references vary by framework |
| Built Project | Output artifact on disk | PBIP tree / JSON file / HTML folder |

---

## Agent Inventory

Agents are either **framework-agnostic** (work with any reporting framework)
or **framework-specific** (implement the contract for a single framework).

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                             AGENT ROSTER                                     │
├──────────────────────┬──────────────┬────────────────────────────────────────┤
│  ADVISORY AGENTS     │  Agnostic    │  report-type-advisor— Goals → Brief   │
│  (pre-pipeline)      │              │  meeting-task-router— Meeting→ Tasks  │
├──────────────────────┼──────────────┼────────────────────────────────────────┤
│  PLANNING AGENTS     │  Agnostic    │  spec-writer        — Reqs → spec.md  │
│  (framework-agnostic)│              │  data-profiler      — Data → Profile  │
│                      │              │  layout-strategist  — Spec → Layouts  │
│                      │              │  ab-evaluator       — Variants→Winner │
├──────────────────────┼──────────────┼────────────────────────────────────────┤
│  BUILD AGENTS        │  Power BI    │  powerbi-engineer   — Builds PBIP     │
│  (framework-specific)│              │  semantic-model-analyzer — Data fixes  │
│                      ├──────────────┼────────────────────────────────────────┤
│                      │  Grafana     │  grafana-engineer   — Builds JSON     │
│                      │  (exp.)      │    dashboards                          │
│                      ├──────────────┼────────────────────────────────────────┤
│                      │  HTML        │  html-engineer      — Builds static   │
│                      │  (exp.)      │    HTML/CSS/JS reports                 │
├──────────────────────┼──────────────┼────────────────────────────────────────┤
│  QUALITY AGENTS      │  Power BI    │  powerbi-tester     — PBIP validation │
│  (framework-specific │  Grafana     │  grafana-tester     — JSON validation │
│   testers; agnostic  │  HTML        │  html-tester        — HTML validation │
│   designer)          │  Agnostic    │  designer           — Visual quality  │
├──────────────────────┼──────────────┼────────────────────────────────────────┤
│  UTILITY AGENTS      │  Agnostic    │  kusto-expert       — KQL queries     │
│                      │              │  QandA              — NL queries      │
├──────────────────────┼──────────────┼────────────────────────────────────────┤
│  COORDINATOR         │  Agnostic    │  report-orchestrator — Pipeline mgmt  │
└──────────────────────┴──────────────┴────────────────────────────────────────┘
```

---

## Agent Deep Dives

### 1. report-orchestrator (Coordinator — Framework-Agnostic)

| Attribute | Detail |
|---|---|
| **Role** | Project manager and pipeline coordinator. Never builds or edits output files directly. |
| **Category** | Coordinator |
| **Framework** | Agnostic — routes to the correct framework-specific engineer and tester |
| **Inputs** | User requirements (verbal or written) |
| **Outputs** | Orchestrated report delivery; progress updates; optional cloud publication |
| **Delegates to** | Every other agent in the roster |
| **Invoked by** | The user (entry point for new report requests) |

The orchestrator owns the entire 10-step pipeline. It gathers requirements
(including framework selection), dispatches work to specialized agents,
interprets their results, manages error budgets (5 fix attempts, 3 design
rounds), and presents the final deliverable. It tracks pipeline state —
current phase, attempt counts, latest test/design results — and keeps the
user informed at every transition.

**Key behaviors:**
- Selects the target framework in Step 0 (defaults to Power BI if not specified)
- Asks focused questions one at a time during intake (never bundles)
- Routes BUILD to the correct framework engineer and TEST to the matching tester
- Auto-tests the first build without prompting; prompts for subsequent iterations
- Passes tester errors and designer suggestions verbatim to the engineer
- Never asks an agent to fix errors AND implement design changes simultaneously
- Updates `spec.md` first when requirements change mid-flight

---

### 2. spec-writer (Planning — Framework-Agnostic)

| Attribute | Detail |
|---|---|
| **Role** | Deep requirements analyst that transforms vague stakeholder input into a precise, structured specification. |
| **Category** | Planning |
| **Framework** | Agnostic — produces a spec that any framework engineer can consume |
| **Inputs** | User requirements summary from the orchestrator's intake phase |
| **Outputs** | `spec.md` — the single source of truth for all downstream agents |
| **Delegates to** | None (may interview the user for gaps) |
| **Invoked by** | report-orchestrator (Step 2) |

The spec-writer interviews the user to fill any gaps the orchestrator's intake
didn't cover. It deep-dives into KPI definitions, maps them to data source
columns, structures page layouts, and codifies the selected theme. The
resulting `spec.md` is the contract — every downstream agent reads it.

**What spec.md contains:**
- `framework:` field (powerbi | grafana | html)
- Audience & purpose
- Data source connection details
- KPI definitions with formulas and data bindings
- Page-by-page layout descriptions (using normalized visual types)
- Filter/slicer specifications
- Theme selection (or custom theme tokens)
- Acceptance criteria

---

### 3. data-profiler (Planning — Framework-Agnostic)

| Attribute | Detail |
|---|---|
| **Role** | Profiles actual data sources before design begins, discovering what the data really looks like. |
| **Category** | Planning |
| **Framework** | Agnostic — data profiling is independent of the output framework |
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
- The layout-strategist uses cardinality data to pick good filter dimensions
- The layout-strategist uses distribution data to choose appropriate chart types
- The framework engineer uses type information and null rates for robust modeling

---

### 4. layout-strategist (Planning — Framework-Agnostic)

| Attribute | Detail |
|---|---|
| **Role** | Generates 2–3 competing layout blueprints using distinct design archetypes, enabling data-driven layout selection. |
| **Category** | Planning |
| **Framework** | Agnostic — uses normalized visual types; framework engineer maps to native visuals |
| **Inputs** | `spec.md` + Data Profile Report (if available) |
| **Outputs** | 2–3 layout blueprints with pixel-precise positions, normalized visual types, data bindings, spacing, and rationale |
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

### 5. ab-evaluator (Planning — Framework-Agnostic)

| Attribute | Detail |
|---|---|
| **Role** | Compares 2–3 layout blueprints side-by-side using a structured scoring framework and recommends the best one. |
| **Category** | Planning |
| **Framework** | Agnostic — evaluates layout quality independent of framework |
| **Inputs** | Layout blueprints from layout-strategist; `spec.md` for context; optional built project files or screenshots for post-build comparison |
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
| Theme Adherence | 1× | Does it follow the selected theme palette? |
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

### 6. powerbi-engineer (Build — Power BI)

| Attribute | Detail |
|---|---|
| **Role** | The Power BI builder. Creates production-ready PBIP projects and semantic models in the PBIR enhanced format. |
| **Category** | Build |
| **Framework** | Power BI only |
| **Inputs** | `spec.md`, winning layout blueprint, fix instructions from tester, design suggestions from designer |
| **Outputs** | Complete PBIP project on disk (`.pbip`, `.pbir`, TMDL, `visual.json` files, DAX measures, theme, etc.) |
| **Delegates to** | None (uses file system tools to create/edit files) |
| **Invoked by** | report-orchestrator (Step 6, fix loops, design loops) — only when `framework: powerbi` |

**What it builds:**
- Full PBIP folder structure (`.pbip`, `.pbir`, `definition/` trees)
- TMDL semantic model (tables, columns, measures, relationships)
- `visual.json` for every visual on every page
- DAX measures for calculated KPIs
- Dark blue theme configuration (or theme from spec)
- Data source connections (Kusto, SharePoint, Excel, SQL)
- Bookmarks, drillthrough pages, conditional formatting
- Multi-table relationships

**Three modes the orchestrator uses it in:**

| Mode | When | Prompt pattern |
|---|---|---|
| **Initial build** | Step 6, first time | "Build a complete PBIP project at `<path>`. Read spec.md. Use this layout blueprint..." |
| **Fix build** | After tester FAIL | "Fix the following errors in the PBIP at `<path>`. The powerbi-tester found..." |
| **Design build** | After designer feedback | "Apply these design improvements to the PBIP at `<path>`. The designer said..." |

**Also handles:**
- Publishing to Microsoft Fabric workspaces
- Refreshing data connections post-publish

---

### 7. semantic-model-analyzer (Build — Power BI)

| Attribute | Detail |
|---|---|
| **Role** | Deep data-layer analyst that detects schema mismatches, type conflicts, and missing transformations that cause import errors or incorrect DAX results. |
| **Category** | Build (optional specialist) |
| **Framework** | Power BI only — analyzes TMDL and M/Power Query |
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

### 8. powerbi-tester (Quality — Power BI)

| Attribute | Detail |
|---|---|
| **Role** | QA validator that checks PBIP projects for structural and runtime correctness. |
| **Category** | Quality |
| **Framework** | Power BI only |
| **Inputs** | PBIP project on disk |
| **Outputs** | Structured pass/fail report with actionable error details and "Errors for Engineer" section |
| **Delegates to** | None (may launch PBI Desktop for Phase 2) |
| **Invoked by** | report-orchestrator (Step 7, fix loop iterations) — only when `framework: powerbi` |

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

### 9. designer (Quality — Framework-Agnostic)

| Attribute | Detail |
|---|---|
| **Role** | Executive dashboard design critic. Reviews visual hierarchy, layout balance, color, typography, data storytelling, and projector readability. |
| **Category** | Quality |
| **Framework** | Agnostic — evaluates design quality regardless of framework; references framework-specific files |
| **Inputs** | Built project on disk + `spec.md` for context; optional screenshots |
| **Outputs** | Prioritized design suggestions (Priority 1, 2, 3) with specific file/property references + letter grade (A–F) |
| **Delegates to** | None (read-only — never modifies files) |
| **Invoked by** | report-orchestrator (Step 8, design loop iterations) |

**What it evaluates:**
- Visual hierarchy and reading order
- Color consistency against the selected theme
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

### 10. grafana-engineer (Build — Grafana, Experimental)

| Attribute | Detail |
|---|---|
| **Role** | Builds Grafana dashboard JSON files from specs and layout blueprints. |
| **Category** | Build |
| **Framework** | Grafana only |
| **Inputs** | `spec.md`, winning layout blueprint, fix instructions from tester, design suggestions from designer |
| **Outputs** | Grafana dashboard JSON file (compatible with Grafana provisioning or API import) |
| **Delegates to** | None |
| **Invoked by** | report-orchestrator (Step 6) — only when `framework: grafana` |

**Experimental** — this agent is under active development. It maps normalized
visual types to Grafana panel types (Stat, Time series, Bar chart, Table, etc.),
configures datasource references, and applies the Grafana Dark theme tokens.

**What it builds:**
- Dashboard JSON with panel layout (gridPos)
- Datasource references and query targets
- Template variables for interactive filtering
- Panel-level overrides and thresholds
- Theme-aligned color configuration

---

### 11. html-engineer (Build — HTML, Experimental)

| Attribute | Detail |
|---|---|
| **Role** | Builds static HTML/CSS/JS report pages from specs and layout blueprints. |
| **Category** | Build |
| **Framework** | HTML only |
| **Inputs** | `spec.md`, winning layout blueprint, fix instructions from tester, design suggestions from designer |
| **Outputs** | HTML folder with `index.html`, CSS, JS, and optional chart library assets |
| **Delegates to** | None |
| **Invoked by** | report-orchestrator (Step 6) — only when `framework: html` |

**Experimental** — this agent is under active development. It maps normalized
visual types to HTML/CSS/JS implementations (Chart.js, D3, or vanilla HTML
tables) and applies the selected theme tokens as CSS custom properties.

**What it builds:**
- Semantic HTML structure (`<main>`, `<section>`, `<article>`)
- CSS Grid or Flexbox layout matching blueprint positions
- Chart.js or D3.js visualizations for charts and gauges
- `<table>` elements for tabular data
- Responsive CSS with theme tokens as custom properties (`--page-background`, etc.)
- Optional JS-based filtering UI

---

### 12. grafana-tester (Quality — Grafana, Experimental)

| Attribute | Detail |
|---|---|
| **Role** | Validates Grafana dashboard JSON for structural correctness. |
| **Category** | Quality |
| **Framework** | Grafana only |
| **Inputs** | Grafana dashboard JSON file on disk |
| **Outputs** | Structured pass/fail report |
| **Invoked by** | report-orchestrator (Step 7) — only when `framework: grafana` |

**Validation checks:**
- JSON schema compliance against Grafana dashboard spec
- Panel `gridPos` overlap detection
- Datasource reference validity
- Template variable consistency

---

### 13. html-tester (Quality — HTML, Experimental)

| Attribute | Detail |
|---|---|
| **Role** | Validates HTML report projects for correctness and accessibility. |
| **Category** | Quality |
| **Framework** | HTML only |
| **Inputs** | HTML project folder on disk |
| **Outputs** | Structured pass/fail report |
| **Invoked by** | report-orchestrator (Step 7) — only when `framework: html` |

**Validation checks:**
- HTML validity and semantic structure
- CSS property validation
- Accessibility audit (WCAG compliance)
- Chart library asset integrity
- Browser render test (if available)

---

### 14. kusto-expert (Utility — Framework-Agnostic)

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

### 15. QandA (Utility — Power BI)

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

### 16. report-type-advisor (Advisory — Framework-Agnostic)

| Attribute | Detail |
|---|---|
| **Role** | Pre-pipeline advisor that helps users decide what *kind* of report to build before entering the orchestration pipeline. |
| **Category** | Advisory (pre-pipeline) |
| **Framework** | Agnostic |
| **Inputs** | User's goals, audience description, data description |
| **Outputs** | Report Brief (YAML) containing recommended report archetype, framework, KPI patterns |
| **Delegates to** | None |
| **Invoked by** | report-orchestrator (Mode 4 — Template Quick Start) or user directly |

**Report Archetypes** (the advisor maps user needs to one of these):

| Archetype | Best For | Typical Framework |
|---|---|---|
| Executive Summary | VP/C-suite overview | Power BI |
| Operational Dashboard | Real-time ops monitoring | Grafana or Power BI |
| Compliance / Audit Report | Policy adherence tracking | Power BI |
| Performance Scorecard | Target vs actual measurement | Power BI |
| Trend Analysis | Time-series investigation | Grafana or Power BI |
| Inventory / Status Tracker | Item lifecycle tracking | Power BI or HTML |
| Financial Report | Revenue/spend/budget analysis | Power BI |
| Customer / User Analytics | Engagement/retention metrics | Power BI |

**When to invoke:**
- User says "I need a report" but doesn't know what type
- User describes a problem/goal rather than a specific report structure
- Orchestrator Mode 4 (Template Quick Start) always starts here

---

### 17. meeting-task-router (Advisory — Framework-Agnostic)

| Attribute | Detail |
|---|---|
| **Role** | Reads Teams meeting summary notes, extracts follow-up tasks, and routes each task to the correct report project for implementation. |
| **Category** | Advisory (task routing) |
| **Framework** | Agnostic |
| **Inputs** | Teams meeting summary (via WorkIQ), list of known report projects |
| **Outputs** | YAML execution plan mapping tasks → projects → actions |
| **Delegates to** | None (produces plan consumed by orchestrator) |
| **Invoked by** | report-orchestrator (Mode 3 — Meeting Follow-Up) |

**Task Categories:**

| Category | Description | Routed To |
|---|---|---|
| `dashboard_change` | Modify existing visual/KPI | Framework-specific engineer |
| `new_visual` | Add new chart/table/card | spec-writer → engineer |
| `data_change` | New data source or query change | data-profiler → engineer |
| `spec_change` | Requirements clarification | spec-writer |
| `investigation` | Data question, ad-hoc analysis | kusto-expert or QandA |
| `external` | Non-report tasks (not routed) | Surfaced to user, not executed |

**Multi-project routing:** A single meeting may generate tasks for different
report projects. The router groups tasks by project and produces a separate
execution plan for each.

---

## Orchestration Pipeline (10 Steps)

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
│  │ Step 0: FRAMEWORK SELECTION                                  │   │
│  │                                                              │   │
│  │  Orchestrator determines the target framework:               │   │
│  │                                                              │   │
│  │  • If user specifies → use that framework                    │   │
│  │  • If not specified  → default to Power BI                   │   │
│  │                                                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │   │
│  │  │ Power BI │  │ Grafana  │  │  HTML    │                   │   │
│  │  │(default) │  │  (exp.)  │  │  (exp.)  │                   │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘                   │   │
│  │       └──────────────┼──────────────┘                        │   │
│  │                      ▼                                       │   │
│  │  Framework choice is recorded in spec.md and used to         │   │
│  │  route BUILD (Step 7) and TEST (Step 8) to the correct       │   │
│  │  framework-specific agents.                                  │   │
│  │                                                              │   │
│  │  Output: Selected framework                                  │   │
│  └──────────────────────┬───────────────────────────────────────┘   │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 1: INTAKE                                               │   │
│  │                                                              │   │
│  │  Orchestrator asks focused questions one at a time:          │   │
│  │  Q1  Who is this for? (VP / Manager / Analyst)               │   │
│  │  Q2  Where's the data? (Kusto / SharePoint / Excel / SQL)    │   │
│  │  Q3  What KPIs matter? (freeform)                            │   │
│  │  Q4  How many pages?                                         │   │
│  │  Q5  Page/canvas height? (720 / 1080 / 1320 / custom)       │   │
│  │  Q6  Visual types? (cards / charts / tables / mix)           │   │
│  │  Q7  Filters & slicers?                                      │   │
│  │  Q8  Color theme? (default per framework, or custom)         │   │
│  │  Q9  Section headers & dividers?                             │   │
│  │  Q10 Output location on disk?                                │   │
│  │  Q11 Publish to cloud? (Fabric / Grafana / manual)           │   │
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
│  │  • Records framework + theme selection    │  • Writes     │  │   │
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
│  │  • Finds cardinality (good filter dims)   │    every col  │  │   │
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
│  │  │ Variant A   │  │ Variant B   │        │ using normal-  │ │   │
│  │  │"Glanceable" │  │"Narrative"  │ ◀──────│ ized visual    │ │   │
│  │  │ KPI-forward │  │ Story-forward│        │ types          │ │   │
│  │  │ Big cards   │  │ Charts+flow │        └────────────────┘ │   │
│  │  └──────┬──────┘  └──────┬──────┘  ┌─────────────┐          │   │
│  │         │                │         │ Variant C   │          │   │
│  │         │                │         │"Cmd Center" │          │   │
│  │         │                │         │ Data-dense  │          │   │
│  │         │                │         └──────┬──────┘          │   │
│  │         ▼                ▼                ▼                  │   │
│  │  Each blueprint has pixel-precise positions,                 │   │
│  │  normalized visual types, data bindings, and rationale       │   │
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
│  │ Step 6: SEMANTIC MODEL ANALYSIS (Optional, Power BI only)    │   │
│  │                                           ┌───────────────┐  │   │
│  │  Orchestrator invokes ──────────────────▶ │  SEMANTIC-    │  │   │
│  │  if complex data layer detected           │  MODEL-       │  │   │
│  │                                           │  ANALYZER     │  │   │
│  │  Detects schema mismatches, generates     │               │  │   │
│  │  M transformation code for the engineer   │  • Analyzes   │  │   │
│  │                                           │    KQL/SQL    │  │   │
│  │  Skipped for Grafana & HTML frameworks    │  • Generates  │  │   │
│  │                                           │    M code     │  │   │
│  │  Output: Data-layer fixes (if needed)     └───────┬───────┘  │   │
│  └──────────────────────┬────────────────────────────┘          │   │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 7: BUILD                                                │   │
│  │                                                              │   │
│  │  Orchestrator routes to the framework-specific engineer:     │   │
│  │                                                              │   │
│  │  ┌─────────────────────────────────────────────────────┐     │   │
│  │  │ framework: powerbi  ──▶  POWERBI-ENGINEER           │     │   │
│  │  │   • Creates PBIP folder structure                   │     │   │
│  │  │   • Writes TMDL, visual.json, DAX measures          │     │   │
│  │  │   • Applies theme from spec                         │     │   │
│  │  ├─────────────────────────────────────────────────────┤     │   │
│  │  │ framework: grafana  ──▶  GRAFANA-ENGINEER (exp.)    │     │   │
│  │  │   • Creates dashboard JSON with panels              │     │   │
│  │  │   • Configures datasources and variables            │     │   │
│  │  │   • Applies Grafana theme tokens                    │     │   │
│  │  ├─────────────────────────────────────────────────────┤     │   │
│  │  │ framework: html     ──▶  HTML-ENGINEER (exp.)       │     │   │
│  │  │   • Creates HTML/CSS/JS project folder              │     │   │
│  │  │   • Builds Chart.js/D3 visualizations               │     │   │
│  │  │   • Applies theme as CSS custom properties          │     │   │
│  │  └─────────────────────────────────────────────────────┘     │   │
│  │                                                              │   │
│  │  Output: Built project on disk                               │   │
│  └──────────────────────┬───────────────────────────────────────┘   │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 8: TEST                                                 │   │
│  │                                                              │   │
│  │  Orchestrator routes to the framework-specific tester:       │   │
│  │                                                              │   │
│  │  ┌─────────────────────────────────────────────────────┐     │   │
│  │  │ framework: powerbi  ──▶  POWERBI-TESTER             │     │   │
│  │  │   Phase 1: JSON schema, TMDL syntax, 20+ gotchas    │     │   │
│  │  │   Phase 2: PBI Desktop integration test              │     │   │
│  │  ├─────────────────────────────────────────────────────┤     │   │
│  │  │ framework: grafana  ──▶  GRAFANA-TESTER (exp.)      │     │   │
│  │  │   JSON schema, gridPos overlap, datasource refs      │     │   │
│  │  ├─────────────────────────────────────────────────────┤     │   │
│  │  │ framework: html     ──▶  HTML-TESTER (exp.)         │     │   │
│  │  │   HTML lint, accessibility, asset integrity           │     │   │
│  │  └─────────────────────────────────────────────────────┘     │   │
│  │                                                              │   │
│  │  ┌─────────┐                                                 │   │
│  │  │  FAIL   │◀── errors found                                 │   │
│  │  │         │                                                 │   │
│  │  │ Loop ◀──┘                                                 │   │
│  │  │ back to    ◀─── max 5 fix attempts                        │   │
│  │  │ Step 7                                                    │   │
│  │  │ (BUILD)    PASS ─────────────────────────────────▶        │   │
│  │  └─────────┘                                                 │   │
│  └──────────────────────┬───────────────────────────────────────┘   │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 9: DESIGN REVIEW                    ┌───────────────┐   │   │
│  │                                          │   DESIGNER    │   │   │
│  │  Orchestrator invokes ─────────────────▶ │  (agnostic)   │   │   │
│  │                                          │               │   │   │
│  │  The designer reviews:                   │  • Layout     │   │   │
│  │  • Visual hierarchy                      │    analysis   │   │   │
│  │  • Color consistency                     │  • Color      │   │   │
│  │  • Typography                            │    audit      │   │   │
│  │  • Data storytelling                     │  • Typography │   │   │
│  │  • Projector readability                 │    audit      │   │   │
│  │                                          │  • Screenshot │   │   │
│  │  ┌─────────┐                             │    review     │   │   │
│  │  │ Grade   │◀── C/D/F or B with ─────────│  • Grades     │   │   │
│  │  │ < B     │    3+ suggestions           │    A-F        │   │   │
│  │  │         │                             └───────┬───────┘   │   │
│  │  │ Loop ◀──┘                                     │           │   │
│  │  │ back to    ◀─── max 3 design rounds ──────────┘           │   │
│  │  │ Step 7                                                    │   │
│  │  │ (BUILD)    Grade A/B ────────────────────────────▶        │   │
│  │  └─────────┘                                                 │   │
│  └──────────────────────┬───────────────────────────────────────┘   │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Step 10: FINAL                                               │   │
│  │                                                              │   │
│  │  ✅ Report Complete!                                         │   │
│  │                                                              │   │
│  │  📁 Location: C:\...\MyReport                                │   │
│  │  🔧 Framework: Power BI / Grafana / HTML                     │   │
│  │  🎨 Theme: (per framework default or custom)                 │   │
│  │  📐 Design Grade: A                                          │   │
│  │  🏆 Layout: "Narrative" (won A/B with 49/55)                 │   │
│  │  ✓  Validation: PASS                                         │   │
│  │                                                              │   │
│  │  → Optionally publish to cloud target:                       │   │
│  │    Power BI → Fabric workspace (via powerbi-engineer)        │   │
│  │    Grafana  → Grafana instance (via grafana-engineer)        │   │
│  │    HTML     → manual deploy or web server                    │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow Between Agents

This diagram shows **what artifacts flow between agents** and in what direction.
The framework-specific engineer and tester are selected by the orchestrator
based on the `framework:` field in `spec.md`.

```
                              spec.md
                          (source of truth)
                     includes framework: field
                                │
                 ┌──────────────┼──────────────────────────┐
                 │              │                           │
                 ▼              ▼                           ▼
          ┌────────────┐ ┌────────────┐          ┌─────────────────┐
          │  data-     │ │  layout-   │          │  FRAMEWORK      │
          │  profiler  │ │  strategist│          │  ENGINEER       │
          └─────┬──────┘ └─────┬──────┘          │  ┌───────────┐ │
                │              │                 │  │ powerbi-  │ │
        Data Profile     Layout Blueprints       │  │ engineer  │ │
           Report         (2-3 variants)         │  ├───────────┤ │
                │              │                 │  │ grafana-  │ │
                └──────┬───────┘                 │  │ engineer  │ │
                       ▼                         │  ├───────────┤ │
                ┌────────────┐                   │  │ html-     │ │
                │    ab-     │                   │  │ engineer  │ │
                │  evaluator │                   │  └───────────┘ │
                └─────┬──────┘                   └────────┬───────┘
                      │                                   │
               Winner + Scorecard              Built Project (on disk)
                      │                                   │
                      │                            ┌──────┴───────┐
                      │                            ▼              ▼
                      │                 ┌─────────────────┐ ┌──────────┐
                      │                 │  FRAMEWORK      │ │ designer │
                      │                 │  TESTER         │ │(agnostic)│
                      │                 │  ┌───────────┐  │ └────┬─────┘
                      │                 │  │ powerbi-  │  │      │
                      │                 │  │ tester    │  │ Design Grade
                      │                 │  ├───────────┤  │ + Suggestions
                      │                 │  │ grafana-  │  │      │
                      │                 │  │ tester    │  │      │
                      │                 │  ├───────────┤  │      │
                      │                 │  │ html-     │  │      │
                      │                 │  │ tester    │  │      │
                      │                 │  └───────────┘  │      │
                      │                 └────────┬────────┘      │
                      │                          │               │
                      │                   Validation             │
                      │                   Report                 │
                      │                          │               │
                      └──────────┬───────────────┴───────────────┘
                                 ▼
                         report-orchestrator
                         (routes everything)
```

### Artifact Registry

Every piece of data that moves between agents:

| Artifact | Produced by | Consumed by | Format |
|---|---|---|---|
| `spec.md` | spec-writer | all pipeline agents | Markdown (includes `framework:` field) |
| Data Profile Report | data-profiler | layout-strategist, orchestrator | Structured text |
| Layout Blueprints (×2–3) | layout-strategist | ab-evaluator, orchestrator | Structured text with pixel specs, normalized visual types |
| Winner Scorecard | ab-evaluator | orchestrator → framework engineer | Structured text |
| Built Project (on disk) | framework engineer | framework tester, designer | PBIP tree / Grafana JSON / HTML folder |
| Validation Report | framework tester | orchestrator → framework engineer (on fail) | Structured pass/fail |
| Design Review | designer | orchestrator → framework engineer (on low grade) | Prioritized suggestions |
| M Transformation Code | semantic-model-analyzer | powerbi-engineer (Power BI only) | M/Power Query code |
| KQL Queries | kusto-expert | user, framework engineer | KQL text |
| DAX Results | QandA | user (Power BI only) | Natural language + tables |

---

## Agent Interaction Matrix

Shows which agents **directly communicate** (via the orchestrator routing
messages between them). ● = primary flow, ○ = optional/conditional flow.
ENG and TST represent the **framework-specific** engineer and tester selected
by the orchestrator (e.g., powerbi-engineer + powerbi-tester for Power BI).

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
  │ fw-engineer   │  ●  │  ●  │     │  ●  │  ●  │  -  │  ○  │  ●  │  ●  │     │     │
  │ sem-analyzer  │  ○  │     │     │     │     │  ○  │  -  │     │     │     │     │
  │ fw-tester     │  ●  │     │     │     │     │  ●  │     │  -  │     │     │     │
  │ designer      │  ●  │  ●  │     │     │     │  ●  │     │     │  -  │     │     │
  │ kusto-expert  │  ○  │     │     │     │     │     │     │     │     │  -  │     │
  │ QandA         │     │     │     │     │     │     │     │     │     │     │  -  │
  └───────────────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘

  Legend: ORC = orchestrator, SPW = spec-writer, DPR = data-profiler,
          LST = layout-strategist, ABE = ab-evaluator,
          ENG = framework engineer (powerbi-/grafana-/html-engineer),
          SMA = semantic-model-analyzer (PBI only),
          TST = framework tester (powerbi-/grafana-/html-tester),
          DES = designer, KUS = kusto-expert, Q&A = QandA
```

**Key relationships explained:**

| Relationship | How it works |
|---|---|
| **orchestrator → spec-writer** | Passes confirmed intake summary; receives spec.md |
| **orchestrator → data-profiler** | Passes data source details from spec; receives profile report |
| **orchestrator → layout-strategist** | Passes spec.md + profile findings; receives 2–3 blueprints |
| **orchestrator → ab-evaluator** | Passes layout blueprints; receives scorecard + winner |
| **orchestrator → framework engineer** | Passes spec + blueprint (build), error list (fix), or design suggestions (polish); receives built project. Routed to the correct engineer based on `framework:` in spec.md |
| **orchestrator → framework tester** | Points tester at built project on disk; receives pass/fail report. Routed to the correct tester based on `framework:` in spec.md |
| **orchestrator → designer** | Points designer at built project + spec; receives grade + suggestions |
| **framework tester → framework engineer** | Tester errors are routed through the orchestrator to the engineer for fixes |
| **designer → framework engineer** | Design suggestions are routed through the orchestrator to the engineer for implementation |
| **semantic-model-analyzer → powerbi-engineer** | M transformation code is routed through the orchestrator to the engineer for integration (Power BI only) |
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
                    │   BUILD (Step 7)    │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   TEST  (Step 8)    │
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
                    │  DESIGN  (Step 9)    │
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

These agents are **not part of the main 10-step pipeline** but can be invoked
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

## Theme Enforcement

Themes are selected during the spec phase (Step 2) and enforced by the
orchestrator in every prompt to the framework engineer and designer. Each
framework has a sensible default theme (see [Default Themes Per Framework](#default-themes-per-framework)
in the Framework Contract above), but the user can request any custom theme
during intake.

The theme is expressed using the [Theme Token Schema](#theme-token-schema)
defined in the Framework Contract. Framework engineers translate tokens into
their native format:

| Framework | Theme Implementation |
|---|---|
| Power BI | `reportTheme.json` + visual-level overrides |
| Grafana | Dashboard JSON `style` properties + panel overrides |
| HTML | CSS custom properties (`--page-background`, `--accent`, etc.) |

**Power BI default — Premium Dark Blue:**

```
  Premium Dark Blue Palette (Power BI default)
  ─────────────────────────────────────────────
  #2A324B  ██████  Page background        (token: page-background)
  #353F5A  ██████  Card surfaces          (token: surface)
  #4A90D9  ██████  Accent / column headers(token: accent)
  #FFFFFF  ██████  Primary text           (token: text-primary)
  #B0B8C8  ██████  Secondary text         (token: text-secondary)
  #4CAF50  ██████  Success / green        (token: success)
  #E8A838  ██████  Warning / amber        (token: warning)
  #FF4444  ██████  Critical / red         (token: critical)
```

---

## Standing Design Rules

These rules are the **Power BI defaults**, always included in prompts to the
powerbi-engineer and designer when `framework: powerbi`. They represent
learned preferences that produce the best results for Power BI executive
dashboards. Other frameworks follow their own conventions.

**Power BI design rules:**

| Rule | Detail |
|---|---|
| **Card titles** | NEVER shown unless user explicitly requests them |
| **Slicer headers** | NEVER shown unless user explicitly requests them |
| **Filters/slicers** | Always at the TOP of the page |
| **No "Filters" label** | Slicers sit directly below title/subtitle with no section header or divider |
| **Table/matrix row backgrounds** | Match the report background color (no alternating rows) |
| **Table/matrix column headers** | Use accent color (`#4A90D9` in default theme), white text, Grow to Fit enabled |
| **Table/matrix grid** | Horizontal lines only (surface color), no vertical lines |
| **Table/matrix totals** | Off by default |
| **Section headers** | Go ABOVE their divider lines, not below |
| **Page heights** | Tight — approximately 30px below the lowest visual; pages can differ in height |

**Cross-framework principles** (apply regardless of framework):

| Principle | Detail |
|---|---|
| **5-second rule** | A VP should grasp the key message within 5 seconds |
| **Visual hierarchy** | Most important KPIs are largest and highest on the page |
| **Consistent spacing** | Use the theme's gutter and margin tokens uniformly |
| **Readable at distance** | KPI values ≥ 24pt; body text ≥ 9pt |
| **Accessible contrast** | Text meets WCAG AA contrast ratio against its background |

---

## Report Type Taxonomy

The system recognizes 8 standard report archetypes. The **report-type-advisor**
maps user needs to one of these; the **spec-writer** uses them to pre-fill KPI
patterns; the **layout-strategist** uses them to select layout archetypes.

| Archetype | Purpose | Key Visual Patterns | Default Framework |
|---|---|---|---|
| **Executive Summary** | High-level status for leadership | Large KPI cards, RAG indicators, trend sparklines, status donut | Power BI |
| **Operational Dashboard** | Real-time ops monitoring | Live metric strip, gauges, queue depth, SLA bars | Grafana or Power BI |
| **Compliance / Audit** | Policy adherence and exception tracking | Pass/fail summary row, exception table, aging histogram | Power BI |
| **Performance Scorecard** | Target vs actual measurement | Target line overlays, ranking tables, delta indicators | Power BI |
| **Trend Analysis** | Time-series investigation | Full-width line charts, moving averages, anomaly markers | Grafana or Power BI |
| **Inventory / Status Tracker** | Item lifecycle tracking | Status breakdown donut, owner bar chart, filterable detail table | Power BI or HTML |
| **Financial Report** | Revenue, spend, budget analysis | Currency KPI cards, variance bars, department matrix | Power BI |
| **Customer / User Analytics** | Engagement and retention metrics | Funnel charts, cohort grids, segment selectors | Power BI |

### Archetype → Layout Mapping

| Archetype | Primary Layout | Secondary Layout |
|---|---|---|
| Executive Summary | Glanceable | Narrative |
| Operational Dashboard | Command Center | Narrative |
| Compliance / Audit | Command Center | Glanceable |
| Performance Scorecard | Narrative | Glanceable |
| Trend Analysis | Narrative | Command Center |
| Status Tracker | Narrative | Command Center |
| Financial Report | Narrative | Glanceable |
| Customer Analytics | Narrative | Command Center |

---

## Report Catalog

The orchestrator maintains awareness of previously built report projects. This
enables:
- **"Update the dashboard I built last week"** — the orchestrator can find and
  load an existing project without the user specifying a path
- **Meeting task routing** — when a meeting generates tasks, the meeting-task-
  router matches tasks to known projects by name, data source, or topic
- **Template reuse** — the orchestrator can suggest "build another report like X"

### Catalog Entry Format

Each known project is tracked with:

```yaml
project:
  name: "Service Health Dashboard"
  path: "C:\\Users\\...\\ServiceHealthDashboard"
  framework: powerbi
  report-type: operational-dashboard
  data-sources:
    - type: kusto
      cluster: "https://..."
      database: "ServiceHealth"
  created: 2024-01-15
  last-modified: 2024-02-20
  spec-path: "spec.md"
```

The catalog is built from project folders the user has worked on. The
orchestrator scans for `spec.md` files in known project directories.

---

## Reference Projects (Power BI)

| Project | Location | Use for |
|---|---|---|
| **ADOWorkItemsReport** | `<your-projects-folder>\ADOWorkItemsReport` | Canonical `.platform` format, `pages.json` schema, `report.json` schema. Opens cleanly in PBI Desktop. |
| **S360Dashboard** | `<your-projects-folder>\S360Dashboard` | 2-page dark-theme report with live Kusto data, complex KQL unions, matrix with 6-level drill-down, `visualContainer/2.8.0` schema. |

When the tester or engineer encounters structural errors, compare against
these known-good projects to identify format discrepancies.
