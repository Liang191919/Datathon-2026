# AlphaForge AI — README

**Team 18:** Kevin Xu, Jeremy Vong, Sheng Hege, Song Ning Lan

AlphaForge AI is a self-explaining, agentic investment-analysis **simulator** built for the PolyFinances Datathon 2026 (National Bank × AWS). For a fictional investor profile it:

- scores how well each of 31 assets fits the profile (Profile-Fit Score, 0–100, nine components);
- compares the portfolio with a profile target on a nine-axis radar and proposes rebalancing steps with their what-if impact;
- checks any proposed position against the profile's constraints and **challenges** conflicting requests (e.g. Robert's "30% in Tesla");
- answers free-text questions ("Ask AlphaForge") through a planner (local mode) or an Amazon Bedrock AgentCore agent (agent mode) that may only use numbers produced by deterministic tools — every figure in an answer is checked against the tool outputs.

Every result carries its evidence (value, source, period), assumptions, limitations, confidence level and the steps/tool calls that produced it.

> **Simulation only.** Fictional profiles and portfolios, public data, no real transactions, not personalized financial advice.

---

## 1. Repository layout

```
workshop/
  README.md                       this file (French: README_FR.md)
  TECHNICAL_DOCUMENTATION.pdf     technical documentation (French: TECHNICAL_DOCUMENTATION_FR.pdf)
  ONE_PAGER.pdf                   one-page summary (French: ONE_PAGER_FR.pdf)
  alphaforge_starter/             starter dataset CSVs + build_dataset.py
  alphaforge-ai/                  the application (paths below are relative to this folder)
alphaforge-ai/
  backend/            Python package "alphaforge" (standard library only at runtime)
    alphaforge/
      data/           bundle loader (fixture | dir: | s3://), placeholder fixtures, correlation fallbacks
      features/       feature engine: returns, risk, technicals, GARCH, regime HMM, Hurst, factors, fundamentals, news
      ingest/         live prices/FX/rates, SEC EDGAR (filings, XBRL, Form 4), central-bank RSS, allow-listed news
      monitoring/     source frequencies (EventBridge Scheduler), alerts
      profiles.py     risk bands, caps, radar targets, persona presets, profile validation
      profile_features.py  investor features: tolerances, needs, component weights
      scoring.py      Profile-Fit Score: 9 components, gates, confidence, rank-stability test
      governance.py   constraint checks for a proposed position or allocation, alternatives
      radar.py        nine-axis portfolio radar vs target
      suggestions.py  rule-based suggestions with what-if impact; top picks
      portfolio.py    weights, covariance-based volatility, risk contributions, stress replay
      reasoning.py    local planner: intents, evidence gathering, citations, reasoning steps
      ask.py          Ask: local mode / AgentCore agent mode, guardrails, figure verification
      agent_tools.py  the 14 tools exposed to the agent (Gateway schema generated from here)
      api/            framework-free router and routes
      handlers/       AWS Lambda entry points (API, Gateway tools, ingest)
    tests/            45 tests, no network
  data_pipeline/      ETL from the starter CSVs to the curated bundle (+ 31 tests)
  frontend/           React 19 + Vite 8 + Tailwind 4 + Recharts 3
  agents/             advisor system prompt, Gateway tool schema, harness config, agent design notes
  infra/iam/          least-privilege IAM policy templates
  scripts/            Lambda packaging, harness-config rendering
  docs/               AWS deployment guide, data contract, deliverables tracker
```

---

## 2. Dependencies

| Component | Requirement | Notes |
|---|---|---|
| Backend runtime | **Python ≥ 3.12** | `backend/pyproject.toml`: `dependencies = []` — standard library only. |
| Backend tests | `pytest >= 8` | optional extra `dev`. |
| AWS code paths | `boto3 >= 1.44` | optional extra `aws`; imported lazily, only for S3, DynamoDB, AgentCore, Bedrock Guardrails, Comprehend, EventBridge. Provided by the Lambda runtime / bundled by `scripts/package_lambda.sh`. |
| Data pipeline | Python (standard library) | imports `backend/alphaforge.features`, so use the same Python ≥ 3.12. |
| Frontend | **Node.js 20.19+ or 22.12+** (required by Vite 8) | react ^19.3, react-router-dom ^7.18, recharts ^3.10, vite ^8.3, tailwindcss ^4.3. |
| Rebuilding the starter dataset (optional) | `pandas`, `yfinance` | only for `alphaforge_starter/build_dataset.py`; the CSVs are already provided. |

No API key is stored anywhere. AWS access uses the default credential chain (SSO, `aws login` or an IAM role).

---

## 3. Data sources

| Source | What is used | Period | Terms |
|---|---|---|---|
| **Starter dataset** (`alphaforge_starter/`, provided by the organizers) | 31 assets (10 Canadian equities, 10 U.S. equities, 4 equity ETFs incl. one ESG-screened, 4 bond ETFs, 1 gold ETF, Bitcoin, Ether) | 2021-11-12 → 2026-11-12, retrieved 2026-11-13 | see `alphaforge_starter/sources_metadata.csv` |
| ↳ Yahoo Finance via `yfinance` | `prices_daily.csv` (40,049 rows: OHLC, close, adjusted close, volume, currency); `corporate_actions.csv` (dividends, splits) | daily | Yahoo terms of service |
| ↳ Bank of Canada Valet API | `fx_usdcad_daily.csv` (FXUSDCAD); policy rate V39079, GoC 2y/10y yields, CPI y/y | daily / monthly | Bank of Canada terms |
| ↳ FRED (St. Louis Fed) | DFF, DGS2, DGS10, CPIAUCSL, UNRATE | daily / monthly | FRED legal terms |
| **Team metadata** (`backend/alphaforge/data/fixtures/assets.json`, hand-curated) | asset group, region, tags (e.g. `high_profile`, `established`, `dividend`), defensive flag, sector-based ESG flags (`fossil_fuels` on ENB.TO, CNQ.TO, XOM; `unscreened_index` on XIU.TO, SPY, QQQ) | static | team assumption — not a third-party ESG rating |
| **Optional live ingest** (network) | Yahoo chart API (prices after the starter window, spliced onto the starter series); BoC + FRED rate/FX updates | after 2026-11-12 | as above |
| ↳ SEC EDGAR | submissions, XBRL company facts (fundamentals, first-filed values), 8-K items, Form 4 insider trades — 18 SEC-registered tickers (ATD.TO, CSU.TO are SEDAR+-only) | since 2021-11-01 | SEC fair-access policy (User-Agent with contact required) |
| ↳ Federal Reserve & Bank of Canada RSS | monetary-policy press releases | rolling | public |
| ↳ GDELT DOC 2.0 / Google News RSS | **headline, publisher, date and link only** (no article text), restricted to an allow-list of 20 publishers, headline must name the company | rolling + backfill around past large moves | respective terms |
| ↳ Amazon Comprehend (optional) | headline sentiment instead of the built-in lexicon | — | AWS |

Without ingest, fundamentals and news are reported as **missing with a reason** (never filled in), and the corresponding score components are left out.

The full processing method is in `alphaforge-ai/docs/data_contract.md` and in `TECHNICAL_DOCUMENTATION.pdf` (§4).

---

## 4. Configuration

Backend settings are environment variables (`backend/alphaforge/config.py` and the modules noted).

| Variable | Default | Purpose |
|---|---|---|
| `ALPHAFORGE_DATA_SOURCE` | `fixture` | `fixture` (placeholder metrics, confidence forced low), `dir:<absolute path>` or `s3://bucket/prefix` (curated bundle) |
| `ALPHAFORGE_DATA_TTL_SECONDS` | `900` | how long a loaded bundle is reused before re-reading (so ingest updates reach the API) |
| `ALPHAFORGE_DEFAULT_WINDOW` | `3y` | metrics window (`3y` or `5y`; a missing window falls back with a note) |
| `ALPHAFORGE_STORAGE` | `local` | `local` (JSON files) or `dynamodb` |
| `ALPHAFORGE_LOCAL_STORE_DIR` | `backend/.localdata` | folder for the local JSON store |
| `ALPHAFORGE_TABLE_PREFIX` | `alphaforge-` | DynamoDB table-name prefix |
| `ALPHAFORGE_AUTH_MODE` | `header` | `header` trusts `X-User-Id` (development only); `cognito` reads JWT claims from API Gateway — **set it to `cognito` on Lambda** |
| `AGENTCORE_HARNESS_ARN` | unset | when set, Ask uses the AgentCore advisor; otherwise the local planner |
| `ALPHAFORGE_AGENT_TOOLS` | `inline` | `inline`: tools are sent with each `InvokeHarness` call and executed inside the API; `gateway`: the harness calls its AgentCore Gateway tools |
| `AGENTCORE_MODEL_ID` | unset | Bedrock model / inference profile overriding the harness model |
| `AGENTCORE_TIMEOUT_SECONDS` | `120` | harness execution limit per call |
| `BEDROCK_GUARDRAIL_ID` / `BEDROCK_GUARDRAIL_VERSION` | unset / `DRAFT` | responsible-use guardrail applied to the question and the answer |
| `ALPHAFORGE_SCHEDULER` | `local` | `eventbridge` applies source-frequency changes to EventBridge Scheduler |
| `ALPHAFORGE_SCHEDULER_GROUP` / `_PREFIX` | `alphaforge-ingest` / `alphaforge-ingest-` | EventBridge schedule group and name prefix |
| `ALPHAFORGE_HTTP_USER_AGENT` | placeholder | User-Agent for ingest requests (SEC EDGAR requires an organisation and contact e-mail) |
| `ALPHAFORGE_SENTIMENT` | lexicon | `comprehend` scores headline sentiment with Amazon Comprehend |
| `ALPHAFORGE_STALE_DAYS` | `10` | confidence drops when market data is older than this |

Frontend (Vite): `VITE_API_BASE` (API URL in production), `VITE_AUTH_MODE` (`header` | `cognito`), `VITE_BASE` (path-proxy prefix), `VITE_ALLOWED_HOST`.

---

## 5. Setup and reproduction

Commands are given for macOS/Linux (bash) and Windows (PowerShell). Start from the repository root (`workshop/`) and run `cd alphaforge-ai` first; all paths below are relative to `alphaforge-ai/`. The starter dataset is already in place at `workshop/alphaforge_starter/`, which is where the pipeline and its tests look by default.

### 5.1 Backend + tests (no network)

```bash
# bash
cd backend
python3.12 -m venv .venv && .venv/bin/pip install pytest
.venv/bin/python -m pytest                       # 45 tests
.venv/bin/python -m pytest ../data_pipeline/tests # 31 tests (builds the bundle from the starter CSVs in a temp dir)
```

```powershell
# PowerShell
cd backend
py -3.12 -m venv .venv; .\.venv\Scripts\pip install pytest
.\.venv\Scripts\python -m pytest
.\.venv\Scripts\python -m pytest ..\data_pipeline\tests
```

Expected: `45 passed` and `31 passed` (verified on 2026-11-15 with Python 3.14).

### 5.2 Build the curated bundle from the starter data

```bash
python data_pipeline/build_bundle.py            # optional: --starter DIR --out DIR ; default out = data_pipeline/out
```

Outputs (each with lineage: source SHA-256, generation time, transform version): `assets.json`, `asset_metrics.json` (3y and 5y windows in CAD, including the calendar-2022 stress return), `correlations.json`, `macro_snapshot.json`, `macro_series.json`, `fx_usdcad.json`, `dividends.json`, `data_quality_report.json`, `prices/<ticker>.json` and `asset_features.json`. The build fails loudly on duplicates, unknown tickers, non-positive prices or currency mismatches.

### 5.3 Run the app locally

```bash
# terminal 1 — API on http://127.0.0.1:8000, on the real bundle
cd backend
ALPHAFORGE_DATA_SOURCE=dir:$(cd ../data_pipeline/out && pwd) .venv/bin/python -m alphaforge.devserver
```

```powershell
# PowerShell equivalent
cd backend
$env:ALPHAFORGE_DATA_SOURCE = "dir:" + (Resolve-Path ..\data_pipeline\out)
.\.venv\Scripts\python -m alphaforge.devserver
```

```bash
# terminal 2 — frontend on http://127.0.0.1:5173 (proxies /api to :8000)
cd frontend
npm install
npm run dev
```

Without `ALPHAFORGE_DATA_SOURCE` the API uses the **placeholder fixture**: the UI shows a banner and every confidence is "low".

If you reach the dev server through a path proxy (for example code-server), start the frontend with `VITE_BASE=/absproxy/5173/ npm run dev`; if it runs under another hostname, add `VITE_ALLOWED_HOST=<host>`.

Use the **Viewing as** selector to switch between Amira, Mateo and Robert (each seeded with a fictional portfolio built to show a different tension) or **My own profile** to open the profile wizard. Robert's demo question is "Should I put 30% of my portfolio in Tesla?".

### 5.4 Optional: live data, filings, fundamentals and news (network)

```bash
cd backend
export ALPHAFORGE_HTTP_USER_AGENT="AlphaForge-AI datathon <contact-email>"   # SEC EDGAR requires a contact
.venv/bin/python -m alphaforge.ingest.run --bundle ../data_pipeline/out \
    --jobs market_data,sec_edgar,official_rss,gdelt                         # ~5 min
.venv/bin/python -m alphaforge.ingest.run --bundle ../data_pipeline/out \
    --jobs news_backfill --max-queries 400                                  # ~10 min, resumable
```

The `features` job always runs last and rebuilds `asset_features.json`. Each run writes `ingest_status.json`.

### 5.5 Optional: Ask with the real AgentCore agent

```bash
cd backend
.venv/bin/pip install boto3
AGENTCORE_HARNESS_ARN=arn:aws:bedrock-agentcore:<region>:<account>:harness/<id> \
AGENTCORE_MODEL_ID=<inference-profile-id> \
ALPHAFORGE_DATA_SOURCE=dir:$(cd ../data_pipeline/out && pwd) \
.venv/bin/python -m alphaforge.devserver
```

Your credentials need `bedrock-agentcore:InvokeHarness` on the harness, and the harness role needs model access. In the default `inline` tool mode the harness pauses on each tool call, the API runs the tool against the signed-in user's profile and resumes the same session. Every call appears in the answer's **Trace** tab, and the figures in the answer are verified against those outputs.

### 5.6 Deploying to AWS

Follow `docs/aws_deployment.md`: S3 (curated bundle, frontend), DynamoDB (5 tables), Lambda (API, tools, ingest), Cognito, API Gateway HTTP API, AgentCore Gateway + Harness, Bedrock Guardrails, CloudFront and EventBridge Scheduler. IAM templates are in `infra/iam/`. Use `scripts/package_lambda.sh` to build the Lambda zip and `scripts/render_harness_config.py` to fill `agents/harness_config.json`.

---

## 6. How results are produced (short version)

- **Numbers come from deterministic code, never from the LLM.** In agent mode every percentage, decimal and x/100 score in the answer is matched against tool outputs; unmatched figures are listed and confidence drops to low.
- **Profile-Fit Score** = weighted sum of nine components (volatility fit, loss & recovery fit, risk-adjusted return, business quality, income, trend & timing, preference fit, diversification contribution, news & events). Weights come from the profile; non-applicable components are dropped and their weight redistributed. Gates: excluded assets are capped at 20; a volatility or loss fit below 30 caps the score at 45.
- **Rank stability**: the ranking is recomputed 300 times with every weight multiplied by a random factor in [0.75, 1.25].
- **Thresholds are labelled assumptions** (bands, tolerances, caps, weights, targets) and are returned with every result.
- **Missing data lowers confidence instead of being filled in.**

See `TECHNICAL_DOCUMENTATION.pdf` (in this folder) for the full method, an end-to-end worked example and the validation evidence.

---

## 7. Status and known gaps

| Part | State |
|---|---|
| Data pipeline, feature engine, scoring, radar, suggestions, governance, stress replay, local Ask | Working, tested (76 tests) |
| Ingest jobs (EDGAR, news, live prices) | Working locally; Lambda handler ready |
| AgentCore agent mode (inline and Gateway tools), guardrails, figure verification | Code and config ready; tested with a fake harness |
| AWS deployment | Guide, IAM templates and packaging scripts ready (see `docs/aws_deployment.md`) |
| Cognito sign-in screen in the frontend | Not built (API side done) |
