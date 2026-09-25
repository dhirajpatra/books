# AGENTS.md
### Multi-Agent AI Platform — Finance & Banking Domain
*(Example manifest: covers investment research, regulatory risk, and compliance use cases)*

This file documents every autonomous/semi-autonomous agent operating in this system: its purpose, scope of authority, data it may touch, tools it may call, and the guardrails it must respect. It is the source of truth for anyone (human or AI) extending the system. If an agent's behavior is not described here, it should not be built without an update to this file first.

---

## 1. System Overview

A multi-agent pipeline that ingests market data, financial statements, macroeconomic indicators, and regulatory corpora, and produces investment research, risk scoring, and compliance narratives for internal analysts (never as direct client-facing advice unless explicitly licensed and reviewed).

- **Orchestration:** LangGraph state machine, with n8n handling scheduled/event-driven workflows and human-in-the-loop checkpoints.
- **Knowledge layer:** Neo4j (graph RAG) for entity/relationship reasoning — counterparties, exposures, regulations, obligations. Vector store for unstructured filings, news, and research notes.
- **Interface layer:** MCP servers expose tools to agents in a controlled, permissioned way (see §4).
- **Deployment:** Local-first for development (docker-compose), portable to Azure/AWS/GCP for production.

---

## 2. Agent Roster

| Agent | Role | Autonomy Level | Escalates To |
|---|---|---|---|
| **Intake/Router Agent** | Classifies incoming request (company analysis, discovery/screening, regulatory query, portfolio review) and routes to the correct sub-graph | Full autonomy on routing only | N/A |
| **Fundamentals Analyst Agent** | Pulls 5–10yr financials, computes ratios, growth/quality scores | Read-only, autonomous | Fundamentals Reviewer (human) on anomalies |
| **Macro/Sector Agent** | Retrieves macro and sector-level indicators (rates, inflation, sector cycles) relevant to the company/sector under review | Read-only, autonomous | — |
| **Risk Scoring Agent (EVT/CCA)** | Applies extreme value theory / contingent claims analysis for tail-risk and systemic-risk scoring | Computation-only, autonomous | Quant Reviewer on model drift or out-of-bound outputs |
| **Regulatory GraphAgent** | Cypher/Text2Cypher/similarity-search queries over the compliance graph (Basel III/IV, Dodd-Frank, DORA, SREP) for gap analysis and contagion-path mapping | Read-only, autonomous | Compliance Officer (human) — always, before any output leaves the system |
| **Discovery/Screening Agent** | When no target company is named, screens the universe against user-stated risk appetite, horizon, and return goals | Autonomous within screening; never auto-executes | User confirmation required to proceed to deep analysis |
| **Portfolio Fit Agent** | Maps a candidate position against the user's existing holdings and stated risk profile | Read-only, autonomous | — |
| **Synthesis/Reporting Agent** | Aggregates outputs from the above into a single research narrative or compliance memo | Autonomous drafting, human sign-off before distribution | Human reviewer, mandatory |
| **Audit/Trace Agent** | Logs every tool call, data source, and model version used per session for reproducibility and regulatory audit trails | Always-on, passive | — |

---

## 3. Data Boundaries

- Agents may only access data sources explicitly registered with the Intake/Router Agent for a given session.
- No agent may combine **client-identifying data** with **model training/fine-tuning pipelines** without a separate, explicitly logged consent step.
- Fictional/sandboxed entities (e.g., synthetic bank names) must be used in all demo, hackathon, or non-production graphs — never real counterparty names outside a licensed production environment.
- All financial figures pulled from third-party MCP data sources must be tagged with source + timestamp in the output; the Synthesis Agent may not present ungrounded numbers.

## 4. Tool / MCP Server Permissions

| Tool Class | Available To | Notes |
|---|---|---|
| Market data MCP (prices, filings) | Fundamentals Analyst, Macro/Sector, Portfolio Fit | Read-only |
| News/macro MCP | Macro/Sector, Discovery/Screening | Read-only |
| Neo4j Cypher Template tools | Regulatory GraphAgent | Fixed, reviewed templates only — no free-form Cypher in production |
| Text2Cypher | Regulatory GraphAgent | Sandboxed; output Cypher is logged and diffed against approved templates before execution |
| Similarity Search (vector) | Regulatory GraphAgent, Fundamentals Analyst | — |
| n8n workflow triggers | Intake/Router only | Other agents cannot self-schedule or self-trigger workflows |

No agent has write access to any external system (brokerage, ledger, filing system) in this manifest. This system is **analysis and advisory only**; execution agents are explicitly out of scope and would require a separate manifest, additional licensing review, and human-in-the-loop trade approval.

## 5. Guardrails (non-negotiable)

1. **No autonomous trade execution.** Ever. This system produces research and risk output, not orders.
2. **No client-facing investment advice without a licensed human reviewer's sign-off.**
3. Regulatory GraphAgent output is **always** routed through a human Compliance Officer before it leaves the system — it is a research aid, not a compliance ruling.
4. Every numeric claim in a final report must be traceable to a specific tool call in the Audit/Trace log.
5. Discovery/Screening Agent must disclose the criteria and data window it used — no "black box" recommendation.
6. Any model or prompt change affecting the Risk Scoring Agent requires a logged review (model risk governance, akin to SR 11-7 / equivalent internal model risk policy).

## 6. Failure Modes & Escalation

- Data source unavailable → Router Agent flags partial-data status in the final report; Synthesis Agent must not silently fill gaps.
- Conflicting outputs between Fundamentals and Risk Scoring Agents → escalate to human reviewer, do not average or auto-resolve.
- Regulatory GraphAgent detects a potential contagion/exposure breach → immediate flag to Compliance Officer, output withheld from standard research channel.

## 7. Change Management

- Adding a new agent, tool, or data source requires a PR against this file plus review sign-off (engineering + compliance).
- Sandboxed/hackathon variants of this system (e.g., synthetic-entity graphs) must be clearly labeled as non-production in this file's changelog.
