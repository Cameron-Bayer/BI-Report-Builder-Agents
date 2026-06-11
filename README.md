# BI Report Builder Agents

A collection of **GitHub Copilot CLI custom agents** that work together as a report &
dashboard "factory." A single orchestrator coordinates specialized planning, engineering,
testing, and design agents to build production-ready reports across multiple frameworks
(**Power BI**, **Grafana**, and **HTML**).

All agents live in [`.github/agents/`](.github/agents/) as `*.agent.md` files.

---

## Quick start

GitHub Copilot CLI auto-discovers agents from `.github/agents/` in the **git repo root**
(or current working directory), and from your global `~/.copilot/agents/` folder.

### Option A — use them in this repo (recommended)

```bash
git clone https://github.com/Cameron-Bayer/BI-Report-Builder-Agents.git
cd BI-Report-Builder-Agents
copilot
```

Then list/select an agent inside Copilot CLI:

```
/agent
```

Most workflows start with the **`report-orchestrator`**, which drives the others.

### Option B — install globally

Copy the agent files into your personal agents folder so they're available in every session:

```bash
# macOS / Linux
cp .github/agents/*.agent.md ~/.copilot/agents/

# Windows (PowerShell)
Copy-Item .github\agents\*.agent.md $HOME\.copilot\agents\
```

---

## Prerequisites

These agents call out to external systems. Depending on which agents you use, you may need:

- **Azure Data Explorer / Kusto** access (data profiling, KQL, live data sources)
- **Microsoft Fabric / Power BI** workspace + permissions (build, publish, refresh, snapshot, Q&A)
- **Power BI Desktop** installed (used by `powerbi-tester` for trace-log validation)
- **Grafana** instance (optional, for `grafana-tester` end-to-end import)
- **Node.js** (for `html-engineer` live-mode proxy server)
- **Microsoft Teams / WorkIQ** access (for `meeting-task-router`)

Agent files do **not** carry credentials or MCP/tool configuration — configure those
separately in your environment (e.g. via `/mcp` in Copilot CLI).

---

## The agents

### Orchestration & intake

| Agent | What it does |
|-------|--------------|
| **report-orchestrator** | End-to-end orchestrator. Four modes: build new reports, edit existing projects, implement Teams meeting follow-ups, and template quick-start. Coordinates all other agents and manages error budgets. |
| **report-type-advisor** | Interviews the user and recommends a report archetype and the best framework (Power BI / Grafana / HTML). Produces a "report brief." Framework-agnostic. |
| **meeting-task-router** | Fetches follow-up tasks from Microsoft Teams meeting summaries and routes them to the correct report/dashboard project. |
| **spec-writer** | Deep requirements analyst. Turns raw input into a comprehensive `spec.md` that all downstream agents consume. |

### Planning & data

| Agent | What it does |
|-------|--------------|
| **data-profiler** | Profiles data sources (Kusto/ADX, SharePoint, Excel, SQL) — distributions, cardinality, outliers, null rates, correlations — to inform KPI and visual choices. |
| **layout-strategist** | Generates 2–3 competing layout blueprints from a spec for A/B comparison. |
| **ab-evaluator** | Scores 2–3 layout or report variants side-by-side and recommends a winner. |
| **semantic-model-analyzer** | Analyzes semantic-model data sources (KQL, SQL, M) for schema mismatches and generates M transformation layers. _(Power BI)_ |
| **kusto-expert** | Expert KQL query builder/analyzer. Connects to ADX clusters, explores schemas, and generates optimized queries. |

### Framework engineers

| Agent | What it does |
|-------|--------------|
| **powerbi-engineer** | Builds production-ready PBIP reports and semantic models (PBIR format, TMDL, DAX). Publishes/refreshes in Microsoft Fabric. |
| **grafana-engineer** | _[Experimental]_ Builds and modifies Grafana dashboard JSON from spec + blueprints. |
| **html-engineer** | _[Experimental]_ Builds responsive HTML/CSS/JS dashboards (Chart.js), static or live-proxy modes. |

### Framework testers

| Agent | What it does |
|-------|--------------|
| **powerbi-tester** | Two-phase validation of PBIP projects (schema/structure + Power BI Desktop trace logs). |
| **grafana-tester** | Validates Grafana dashboard JSON; optionally imports to a running instance. |
| **html-tester** | Validates HTML/CSS/JS reports for correctness, accessibility, and visual quality. |

### Design, publish & query

| Agent | What it does |
|-------|--------------|
| **designer** | Executive dashboard design reviewer. Produces prioritized, actionable design suggestions (does not modify files). |
| **powerbi-snapshot** | Captures PNG snapshots of Power BI report pages/visuals via the Power BI REST API. |
| **QandA** | Interactive Q&A over Power BI reports — generates and runs DAX against a selected semantic model. |

---

## Typical flow

```
report-type-advisor  →  spec-writer  →  data-profiler
        │                    │
        ▼                    ▼
 layout-strategist  →  ab-evaluator
        │
        ▼
 framework engineer  ⇄  framework tester      (powerbi / grafana / html)
        │
        ▼
     designer  →  publish / snapshot  →  QandA
```

The **report-orchestrator** drives this pipeline and manages handoffs between agents.

---

## Notes

- `architecture.md` (in `.github/agents/`) documents the shared multi-framework contract
  referenced by the engineer agents.
- Agents marked **[Experimental]** (Grafana, HTML) are less mature than the Power BI path.
