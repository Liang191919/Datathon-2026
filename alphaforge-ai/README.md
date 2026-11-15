# AlphaForge AI

AlphaForge AI is an explainable investment-analysis simulator for the PolyFinances Datathon 2026 (National Bank × AWS). It scores how well each asset fits a fictional investor profile, compares the portfolio with a target on a radar chart, suggests rebalancing steps with their what-if impact, and challenges requests that conflict with the profile. Every answer shows its evidence, assumptions, limits and the steps that produced it.

> **Simulation only.** Fictional profiles and portfolios, public data, no real transactions, not personalized financial advice.

## Status

| Part | State |
|---|---|
| Feature engine (`backend/alphaforge/features`): returns, volatility, drawdowns, Sharpe/Sortino, beta, RSI, SMA, MACD, ATR, liquidity, fundamentals, factor exposures, GARCH, regimes, Hurst, news and insider features | Working, with tests. Built by the pipeline and rebuilt after every ingest run |
| Fit score and ranked asset list (`scoring.py`, `profile_features.py`): nine components, profile-dependent weights, risk and exclusion gates, rank-stability test | Working, with tests |
| Ask AlphaForge reasoning (`reasoning.py`): plan, gather market and news evidence, evaluate, governance, challenge, answer with citations | Working in local mode, with tests; agent mode uses the same tools and contract |
| Ingest jobs (`backend/alphaforge/ingest`): live prices and rates, SEC EDGAR filings, XBRL fundamentals and Form 4, central-bank releases, allow-listed news and a news backfill around past price moves | Working locally; Lambda handler `alphaforge.handlers.ingest_lambda.handler` ready, not deployed |
| Radar, suggestions, governance, stress replay | Working, with tests |
| API (local dev server and AWS Lambda handler) | Working locally; Lambda handler covered by tests |
| React app: dashboard, profile wizard, ranked assets, asset detail with features and news, Ask with reasoning, monitoring, data and limits | Working locally |
| Advisor agent (AgentCore Harness + Gateway tools) | Code and config ready; not deployed yet (`docs/aws_deployment.md`) |
| Data pipeline from the starter CSVs | Working, with tests (`data_pipeline/`). The backend uses it when `ALPHAFORGE_DATA_SOURCE` points to `data_pipeline/out`; the default is still the placeholder fixture. Contract: `docs/data_contract.md` |
| Cognito sign-in screen | Not built (API side done) |

## Repository layout

```
backend/        Python package "alphaforge" (no third-party runtime dependencies)
  alphaforge/
    data/         curated-bundle loader (fixture, folder or S3) + placeholder fixtures
    features/     feature engine: indicators, market data alignment, fundamentals, news features
    ingest/       ingest jobs: live market data, SEC EDGAR, news, central banks; allow-listed sources
    profiles.py   risk bands, caps, radar targets, persona presets
    profile_features.py how a profile is read: tolerances, needs, component weights
    scoring.py    Profile-Fit Score 0-100: nine components, gates, rank stability, evidence
    reasoning.py  Ask AlphaForge planner: intents, evidence gathering, citations, reasoning steps
    radar.py      nine-axis portfolio radar vs target
    suggestions.py rules on radar gaps and constraint flags, what-if impacts, top picks
    governance.py constraint checks for a proposed position or allocation
    ask.py        "Ask AlphaForge": local mode and AgentCore agent mode, figure verification
    agent_tools.py tools exposed to the agent (Gateway schema generated from here)
    api/          framework-free router and routes
    handlers/     Lambda entry points (API, Gateway tools, ingest)
    monitoring/   sources, adjustable frequency (EventBridge Scheduler), alerts
  tests/
frontend/       React + Vite + Tailwind + Recharts
agents/         advisor system prompt, Gateway tool schema, harness config, design notes
infra/iam/      least-privilege IAM policy templates
scripts/        Lambda packaging, harness config rendering
data_pipeline/  ETL for the starter CSVs (to build)
docs/           AWS deployment guide, data contract, deliverables tracker
```

## Run locally

Requirements: Python 3.12+ and Node.js 20.19+ (or 22.12+), which Vite 8 needs.

```bash
# Backend API on http://127.0.0.1:8000
cd backend
python3.12 -m venv .venv && .venv/bin/pip install pytest   # or: uv venv --python 3.12 .venv && uv pip install --python .venv/bin/python pytest
.venv/bin/python -m pytest                                 # 44 tests, no network
.venv/bin/python -m alphaforge.devserver                   # placeholder metrics

# Real data from ../alphaforge_starter instead (run from backend/)
python3 ../data_pipeline/build_bundle.py                   # writes ../data_pipeline/out, including asset_features.json
.venv/bin/python -m pytest ../data_pipeline/tests          # 31 tests
ALPHAFORGE_DATA_SOURCE=dir:$(cd ../data_pipeline/out && pwd) .venv/bin/python -m alphaforge.devserver

# Optional, needs network: data after the starter window, filings, fundamentals and news
export ALPHAFORGE_HTTP_USER_AGENT="AlphaForge-AI datathon <your-email>"   # SEC EDGAR requires a contact
.venv/bin/python -m alphaforge.ingest.run --bundle ../data_pipeline/out \
  --jobs market_data,sec_edgar,official_rss,gdelt          # about 5 minutes
.venv/bin/python -m alphaforge.ingest.run --bundle ../data_pipeline/out --jobs news_backfill --max-queries 400   # about 10 minutes, resumable

# Frontend on http://127.0.0.1:5173 (second terminal)
cd frontend
npm install
npm run dev
```

### Ask with the real AgentCore agent

Ask switches from the local engine to the Amazon Bedrock AgentCore Harness when `AGENTCORE_HARNESS_ARN` is set. AWS credentials come from the default chain (`aws login`, SSO or an IAM role). No API key is used, so none goes in the repository.

```bash
cd backend
uv pip install --python .venv/bin/python boto3              # only needed for agent mode
AGENTCORE_HARNESS_ARN=arn:aws:bedrock-agentcore:<region>:<account>:harness/<harness-id> \
AGENTCORE_MODEL_ID=global.anthropic.claude-sonnet-5-5 \
ALPHAFORGE_DATA_SOURCE=dir:$(cd ../data_pipeline/out && pwd) \
.venv/bin/python -m alphaforge.devserver
```

Your credentials need `bedrock-agentcore:InvokeHarness` on the harness. The harness execution role needs model access for the chosen model. In the default `inline` tool mode, the harness pauses on each tool call and the API runs the tool against the signed-in user's profile. The API then sends the result back on the same session. Every call appears in the answer's Trace tab, and the numbers in the answer are checked against those tool outputs.

Use the **Viewing as** selector to switch between the three personas from the brief. Each starts with a fictional portfolio built to show a different tension. **My own profile** opens the wizard.

If you reach the dev server through a path proxy (for example code-server), start it with `VITE_BASE=/absproxy/5173/ npm run dev`. If it runs under another hostname, add `VITE_ALLOWED_HOST=<host>`.

## Configuration

Backend settings come from environment variables (`backend/alphaforge/config.py`):

| Variable | Default | Purpose |
|---|---|---|
| `ALPHAFORGE_DATA_SOURCE` | `fixture` | `fixture`, `dir:/abs/path` or `s3://bucket/prefix` (curated bundle) |
| `ALPHAFORGE_STORAGE` | `local` | `local` (JSON files in `backend/.localdata`) or `dynamodb` |
| `ALPHAFORGE_AUTH_MODE` | `header` locally, `cognito` on Lambda | `header` trusts `X-User-Id` (dev only); `cognito` uses JWT claims from API Gateway |
| `ALPHAFORGE_DEFAULT_WINDOW` | `3y` | metrics window |
| `AGENTCORE_HARNESS_ARN` | unset | when set, Ask uses the AgentCore advisor; otherwise local mode |
| `ALPHAFORGE_AGENT_TOOLS` | `inline` | `inline`: the AlphaForge tools are sent as inline functions with each `InvokeHarness` call and run inside the API (no Gateway needed; shell, file and browser tools are not allowed). `gateway`: the harness uses its AgentCore Gateway tools |
| `AGENTCORE_MODEL_ID` | unset | Bedrock model or inference profile that overrides the harness model, e.g. `global.anthropic.claude-sonnet-5-5` |
| `AGENTCORE_TIMEOUT_SECONDS` | `120` | harness execution limit per call |
| `BEDROCK_GUARDRAIL_ID`, `BEDROCK_GUARDRAIL_VERSION` | unset | responsible-use guardrail on questions and answers |
| `ALPHAFORGE_SCHEDULER` | `local` | `eventbridge` to apply frequency changes to EventBridge Scheduler |
| `ALPHAFORGE_DATA_TTL_SECONDS` | `900` | how long a loaded bundle is reused before it is read again (so ingest updates reach the API) |
| `ALPHAFORGE_HTTP_USER_AGENT` | placeholder | User-Agent for ingest requests; SEC EDGAR requires an organisation and contact address |
| `ALPHAFORGE_SENTIMENT` | `lexicon` | `comprehend` scores headline sentiment with Amazon Comprehend (AWS) |
| `ALPHAFORGE_STALE_DAYS` | `10` | confidence drops when market data is older than this |

Frontend: `VITE_API_BASE` (API URL in production), `VITE_AUTH_MODE` (`header` or `cognito`).

## How results are produced

- **Numbers come from deterministic code**, never from an LLM. In agent mode, the API checks every percentage, decimal and x/100 score in the answer against the tool outputs, and lists any figure it cannot match.
- **Two layers of features.** Asset features (`asset_features.json`) describe each asset; investor features (`profile_features.py`) read the profile into tolerances (volatility band, loss tolerance = the lower of the risk and horizon limits, 2022-type stress tolerance, recovery-time budget), needs (income, quality, ESG, growth) and component weights.
- **Profile-Fit Score** = weighted sum of nine components: volatility fit, loss and recovery fit, risk-adjusted return, business quality, income, trend and timing, preference fit (including ESG controversy evidence from news), diversification contribution and news and events. Weights start from the objective and are adjusted by horizon and needs, with the reason for each adjustment shown. Components that do not apply (fundamentals for a fund) are left out and their weight shared. Excluded assets are capped at 20; a volatility or loss fit below 30 caps the score at 45.
- **Rank stability.** The ranking is recomputed 300 times with each weight multiplied by a random factor from 0.75 to 1.25; each asset shows how often it stays in the top five.
- **News explains, and counts a little.** Filings, central-bank releases and allow-listed headlines are cited in answers and on asset pages. In the score they enter only through the news component and ESG controversy evidence. Moves are matched to articles published around them, never presented as their cause.
- **Thresholds are labelled assumptions** (volatility bands, tolerances, caps, weights, radar targets). They are returned with every result and listed in `backend/alphaforge/profiles.py` and `profile_features.py`.
- **Missing data lowers confidence instead of being filled in**, for example placeholder metrics, a missing window, or correlations estimated from asset-group priors.

## Data

The optional starter dataset (`../alphaforge_starter`) covers 31 assets from 2021-11-12 to 2026-11-12: Yahoo Finance prices via yfinance, plus Bank of Canada and FRED macro series. Sources, terms and caveats are in `alphaforge_starter/README.md` and `sources_metadata.csv`. The fixture metrics are **placeholders** (rounded exploratory figures). `data_pipeline/build_bundle.py` computes the real ones; `docs/data_contract.md` documents the method and its choices.

## Deploying to AWS

Follow `docs/aws_deployment.md`. It covers S3, DynamoDB, Lambda, Cognito, API Gateway, AgentCore Gateway and Harness, Bedrock Guardrails, CloudFront and EventBridge Scheduler.
