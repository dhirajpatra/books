# CLAUDE.md
Guidance for Claude Code (and any AI coding agent) working in this repository.
*(Example: multi-agent finance/banking research & risk platform)*

## Project Shape

This repo implements a multi-agent research and risk-analysis pipeline for the finance/banking domain. See `AGENTS.md` for the full agent roster, data boundaries, and guardrails — **read it before modifying any agent's tools, prompts, or permissions.**

Core stack:
- **Orchestration:** LangGraph (Python) for the agent state machine; n8n for scheduled/event-driven workflows and human-in-the-loop checkpoints
- **Knowledge graph:** Neo4j (Aura in prod, local Neo4j in dev) — Cypher Template tools, Text2Cypher, vector similarity search
- **Vector store:** for unstructured filings, news, research notes (RAG)
- **Agent-tool interface:** MCP servers, one per external data domain (market data, macro/news, regulatory corpus)
- **Deployment:** docker-compose locally → Azure/AWS/GCP in production

## Setup

```bash
docker-compose up -d          # brings up local Neo4j, vector store, n8n
cp .env.example .env          # fill in API keys for market-data / news MCP servers
pip install -r requirements.txt --break-system-packages   # if working outside a venv
```

## Common Commands

```bash
# Run the full agent graph against a sample query
python run_graph.py --query "Analyze company X for a 5yr horizon"

# Run just the Regulatory GraphAgent against the Neo4j sandbox graph
python agents/regulatory_graphagent.py --mode text2cypher --query "..."

# Run tests
pytest tests/ -v

# Lint
ruff check .
```

*(Replace with actual project scripts — this is illustrative.)*

## Code Conventions

- One file per agent under `agents/`; shared tool wrappers under `tools/`.
- Every agent function that calls an external MCP tool must log: tool name, input, timestamp, and source — this feeds the Audit/Trace Agent described in `AGENTS.md`. Don't bypass this logging for "quick" scripts.
- Cypher used by the Regulatory GraphAgent lives in `graph/templates/` as named, version-controlled `.cypher` files. Text2Cypher output is diffed against these templates before execution in anything beyond a local sandbox — don't wire raw LLM-generated Cypher directly to Neo4j Aura in production code paths.
- Prompts live in `prompts/<agent_name>/` as versioned files, not inlined in Python — makes prompt changes reviewable in diffs.

## Domain-Specific Guardrails (apply to any code you write here)

- **No execution logic.** Don't add code paths that place orders, move funds, or write to any brokerage/ledger system. This is a research/analysis system only. If a task seems to need execution, flag it rather than implementing it.
- **No real counterparty/client data in sandbox, demo, or test code.** Use the fictional entities already established in the graph (see `AGENTS.md` §3). Never hardcode or fetch real client-identifying data into test fixtures.
- **Numeric claims need provenance.** Any function producing a figure that ends up in a report must return its source and timestamp alongside the number — don't strip that metadata for convenience.
- **Model risk changes are reviewed, not silent.** Changes to the Risk Scoring Agent (EVT/CCA logic) or its thresholds need an entry in `CHANGELOG_MODEL_RISK.md`, not just a commit message.
- **Human-in-the-loop checkpoints in n8n are not optional steps to "streamline."** If a workflow currently pauses for compliance or fundamentals-reviewer sign-off, don't refactor that into a fully automated path without an explicit, separate request and sign-off.

## Testing Expectations

- New agent logic needs a test with at least one "missing data" case and one "conflicting output from another agent" case — these are the two failure modes called out in `AGENTS.md` §6.
- Regulatory GraphAgent changes should be tested against the sandbox graph (fictional banks) before any Aura production graph is touched.

## What Claude Should Ask Before Doing

- Before adding a new external MCP server or tool permission for any agent — check `AGENTS.md` §4; this needs a manifest update, not just a code change.
- Before changing what data an agent can read/write — check `AGENTS.md` §3 (data boundaries).
- Before "simplifying" a human-review step out of a workflow — don't; ask first.

## Non-Goals

- This is not a trading/execution system.
- This is not a licensed robo-advisor; outputs are internal research aids pending human review.
- Don't build client-facing chat surfaces on top of this without a separate compliance review — out of scope for this repo.
