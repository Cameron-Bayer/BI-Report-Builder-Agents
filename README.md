# Copilot Agent Collections

A collection of **GitHub Copilot custom agents** for BI reporting, data exploration, and productivity. Each folder contains a self-contained set of agents — `cd` into the one you want and start Copilot.

## What Are These?

These are `.agent.md` files that live in `.github/agents/` directories. When you run **GitHub Copilot** (CLI or VS Code) from a folder containing these files, Copilot automatically discovers them and gains specialized capabilities — like building Power BI reports, writing KQL queries, or answering questions about your data.

**You don't need to configure anything.** Just navigate to the right folder and start talking to Copilot.

## Prerequisites

1. A **GitHub Copilot** subscription (Individual, Business, or Enterprise)
2. One of the following:
   - [**GitHub Copilot CLI**](https://docs.github.com/en/copilot/using-github-copilot/using-github-copilot-in-the-command-line) (terminal)
   - **VS Code** with the [GitHub Copilot Chat](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat) extension

## Getting Started

```bash
# 1. Clone the repo
git clone <this-repo-url>

# 2. Navigate to the agent collection you want
cd BI-Report-Builder

# 3. Launch Copilot
copilot
```

Or open the folder in **VS Code** — Copilot Chat will auto-discover the agents.

---

## Agent Collections

### [`BI-Report-Builder/`](./BI-Report-Builder)

**Full-service report and dashboard factory.** An orchestrator coordinates 15+ specialized agents to build, test, and design reports end-to-end.

| Capability | Description |
|------------|-------------|
| **Build reports from scratch** | Describe what you need → spec → layout → build → test → design review |
| **Edit existing reports** | Make scoped modifications to Power BI, Grafana, or HTML projects |
| **Meeting follow-ups** | Parse Teams meeting action items and route them to the right reports |
| **Template quick start** | Pick a pre-built template to skip most intake Q&A |
| **Q&A on your data** | Ask natural-language questions — get answers via DAX against your semantic models |
| **KQL / Kusto queries** | Explore Azure Data Explorer clusters, build and run optimized KQL |
| **Power BI → PowerPoint** | Generate PPTX files with live, interactive Power BI visual connections |

**Supported frameworks:** Power BI (full) · Grafana (experimental) · HTML/CSS/JS (experimental)

**Agent lineup:**

```
report-orchestrator          — project manager, coordinates the pipeline
├── report-type-advisor      — helps you decide what kind of report to build
├── spec-writer              — deep requirements analysis → spec.md
├── data-profiler            — profiles data sources for distributions & KPI candidates
├── layout-strategist        — generates competing layout blueprints
├── ab-evaluator             — compares layouts, picks the winner
├── powerbi-engineer         — builds/modifies PBIP project files
├── powerbi-tester           — validates PBIP projects for correctness
├── grafana-engineer         — builds/modifies Grafana dashboard JSON
├── grafana-tester           — validates Grafana dashboards
├── html-engineer            — builds/modifies HTML/CSS/JS reports
├── html-tester              — validates HTML reports
├── designer                 — reviews visual quality and executive readability
├── semantic-model-analyzer  — checks data source queries for schema mismatches
├── meeting-task-router      — parses meeting follow-ups → routes to projects
├── QandA                    — natural-language Q&A against Power BI semantic models
├── kusto-expert             — KQL query builder and Kusto cluster explorer
└── powerbi-to-pptx          — generates PPTX with live Power BI visuals
```

**Example prompts:**
- *"Build a Power BI sales dashboard with revenue by region and monthly trends"*
- *"What were our top 10 customers by revenue last quarter?"* (Q&A mode)
- *"Connect to my Kusto cluster and show me error rates for the last 24 hours"*
- *"Create a PowerPoint from my Sales Overview report"*

---

### [`TestAgents/`](./TestAgents)

**Lightweight / experimental agent setup.** Contains the report-orchestrator agent for testing and development purposes.

**Agents:** `report-orchestrator`

---

## Adding New Agent Collections

To add a new set of agents:

1. Create a new folder at the repo root (e.g., `DevOps-Agents/`)
2. Add a `.github/agents/` directory inside it
3. Drop your `.agent.md` files in there
4. Update this README with a description of the new collection

```
your-new-collection/
  .github/
    agents/
      my-agent.agent.md
      another-agent.agent.md
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Agent not found | Make sure you `cd` into the **collection folder** (not the repo root) before launching Copilot |
| "No agents available" | Ensure your Copilot subscription is active and the CLI/extension is up to date |
| Agent seems generic | Confirm the `.github/agents/` folder exists in your current directory |
