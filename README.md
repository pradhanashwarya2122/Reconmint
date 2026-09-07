<pre align="center">
★ ✦ ★ ✦ ★ ✦ ★ ✦ ★ ✦ ★ ✦ ★ ✦ ★ ✦ ★ ✦ ★ ✦ ★

 ____                      __  __ _       _
|  _ \ ___  ___ ___  _ __ |  \/  (_)_ __ | |_
| |_) / _ \/ __/ _ \| '_ \| |\/| | | '_ \| __|
|  _ &lt;  __/ (_| (_) | | | | |  | | | | | | |_
|_| \_\___|\___\___/|_| |_|_|  |_|_|_| |_|\__|

    ★ Payment Reconciliation, Automated ★
★ ✦ ★ ✦ ★ ✦ ★ ✦ ★ ✦ ★ ✦ ★ ✦ ★ ✦ ★ ✦ ★ ✦ ★
</pre>

**An AI Finance Controller agent for the books and the cash.** Solo submission to the **Razorpay AI Buildathon 2026 · Track 4**.

<p align="center">
  <a href="https://reconmint.pages.dev">
    <img alt="Live app" src="https://img.shields.io/badge/%E2%86%92%20Try%20the%20live%20app-reconmint.pages.dev-1F2A1A?style=for-the-badge&labelColor=1F2A1A&color=4B7B4E" height="42">
  </a>
  &nbsp;
  <a href="https://youtu.be/l8uax4pVOZ0">
    <img alt="Watch the demo on YouTube" src="https://img.shields.io/badge/%E2%96%B6%20Watch%20the%205%20min%20demo-YouTube-FF0000?style=for-the-badge&logo=youtube&logoColor=white&labelColor=1F2A1A" height="42">
  </a>
  &nbsp;
  <a href="https://github.com/pradhanashwarya2122/Reconmint">
    <img alt="Source on GitHub" src="https://img.shields.io/badge/Source%20on%20GitHub-pradhanashwarya2122%2FReconmint-1F2A1A?style=for-the-badge&logo=github&logoColor=white&labelColor=1F2A1A&color=6B7660" height="42">
  </a>
</p>

![ReconMint landing page](docs/img/landingpage.png)

> **ReconMint is a two-verifier reconciliation agent.** One guards the AI: any rupee the LLM writes that isn't grounded in the audit table gets rejected. One guards the source data: every exception is appealed live to `api.razorpay.com` to catch stale merchant CSVs. **The Razorpay API is not an integration in this project. It's the second verifier.**

Every Indian merchant on Razorpay lives with three ledgers that never quite agree: their orders, Razorpay's settlement report, and their bank statement. Fees, GST, TCS, T+2 timing, chargebacks. All of it drifts. Most merchants still reconcile it by hand in a spreadsheet, once a month, after the fact.

ReconMint is an **agent that runs the loop for them.** Seven sub-agents cooperate on the batch. Each one owns a decision, branches per record, and writes its reasoning to an audit table. The LLM is on a short leash: it can parse a question and phrase an answer, but a verifier vetoes any rupee it can't ground in the audit trail.

**The AI never touches a number it can't prove.** That's the whole product.

## What's actually unusual about this build

Reconciliation is common. Most of what ReconMint does is table stakes. Four things are not:

- **Truth-Anchor Agent (the Razorpay API as a verifier).** Every exception the engine surfaces is appealed live to `api.razorpay.com` for that exact payment id. If the merchant's uploaded CSV disagrees with what Razorpay actually returns on gross, fee, or tax, the drift is flagged as a stale export. No amount of three-way file matching could ever catch a stale source.
- **Hallucination verifier with magnitude compare.** Every rupee the LLM tries to state is regex-extracted and magnitude-checked against the audit table. If the number doesn't match a real magnitude, the LLM's phrasing is rejected and the deterministic answer is used instead. Signed compare would false-reject correct chargebacks (documented in `FAILURES.md`).
- **Repair Agent with per-record branching.** For every unmatched settlement, three rescue strategies are tried in order (normalize UTR, widen date window, fuzzy amount). First one to clear 85% confidence wins. Every attempt is logged with score and verdict. This is real agent behavior, not a pipeline with an LLM in it.
- **149,250 rows in 217 seconds.** Track 4 asks for 50+ record batches. Benchmark verified at ~3,000x that scale, 687 rec/s sustained. Numbers are in `scripts/benchmark.py` and reproducible from the CLI.

---

## Run it locally in five minutes

You need Python 3.11+ and Node 18+. That's it. No Docker, no cloud account, no database to provision.

**1. Clone and install**

```bash
git clone https://github.com/pradhanashwarya2122/reconmint.git
cd reconmint
pip install -r requirements.txt
cd frontend && npm install && cd ..
```

**2. Add your API keys**

```bash
cp .env.example .env
```

Open `.env` and paste:

- **OPENAI_API_KEY** — get one at [platform.openai.com/api-keys](https://platform.openai.com/api-keys). Any key with a few dollars of credit is enough; the whole demo costs pennies.
- **RAZORPAY_KEY_ID** and **RAZORPAY_KEY_SECRET** — grab TEST-MODE keys from [dashboard.razorpay.com/app/website-app-settings/api-keys](https://dashboard.razorpay.com/app/website-app-settings/api-keys) after switching to Test Mode in the top-right. Test keys start with `rzp_test_`.

Both are optional. If you leave them out, the reconciliation loop still runs; only the LLM-powered explanations and the Truth-Anchor Agent's live appeals are disabled, and each of those degrades to a clear "not configured" state rather than crashing.

**3. Start both servers (two terminals)**

```bash
# terminal one, from the repo root
python -m uvicorn backend.api.main:app --port 8000 --reload
```

```bash
# terminal two, from the repo root
cd frontend && npm run dev
```

**4. Open** [`http://localhost:5173`](http://localhost:5173), click **Try sample data**, and watch every agent light up in real time. Or drop three of your own CSVs into the Upload page.

If port 5173 is taken, Vite will land on 5174 or 5175 automatically and print the URL.

**Don't have your own CSVs?** Grab a sample batch (orders + settlement + bank) from Google Drive: **[Download sample files](https://drive.google.com/drive/folders/1P3ob4rXyFbA94IAP4DG4Oi_9xi8wpQeE?usp=sharing)**. Drop all three into the Upload page and hit Reconcile. These are the same files used in the video demo, so you can follow along step for step.

---

## The agent cast

Not a pipeline. A **cast of small agents**, each with one job and the authority to branch on what it sees.

![ReconMint architecture: sources on the left, eight-agent cast in the middle, outputs on the right, audit trail underneath everything](docs/img/architecture.png)

**Repair Agent is the interesting one.** For every unmatched record it tries three rescue strategies in order (normalize UTR, widen date window, tight fuzzy scoring). First strategy to clear 85% confidence wins. Every attempt is written to `strategy_attempts_json` per record. Open any exception's **Decisions** tab and you see the actual tree for that specific payment: which strategy was tried, what it scored, what it decided, why.

![Repair Agent decision tree for one exception showing three strategies attempted with confidence scores](docs/img/repair-agent.png)

That per-record branching is why this is an agent, not a pipeline.

**Truth-Anchor Agent is the second verifier.** Alongside the hallucination verifier that guards what the LLM says, this one guards the source data itself. For every exception, it appeals to Razorpay's live API and treats the API's answer as ground truth. If the merchant's CSV disagrees on gross, fee, or tax, the agent surfaces the drift line by line. Catches stale merchant exports that no amount of three-way file matching could reveal.

![Truth-Anchor Agent card showing live diff between CSV and Razorpay API for one payment](docs/img/truth-anchor.png)

---

## What each agent does

- **Razorpay-check Agent** hits `api.razorpay.com` at ingest, captures HTTP status + latency + `X-Razorpay-Request-Id`, and confirms schema fidelity live. If keys are missing it says so instead of pretending.
- **Ingest Agent** reads CSV or Excel, detects header rows through preamble, fuzzy-maps 60+ common column aliases to the canonical schema, synthesizes optional columns as zero when absent.
- **Match Agent** reconstructs the fee schedule (MDR + GST-on-MDR + TCS) from gross, then joins on UTR + paise-exact amount + same IST day.
- **Fuzzy Agent** re-scores what Match missed with an amount-bucket index (kept it near-linear at scale).
- **Repair Agent** does per-record branching. Three strategies, first-wins, full audit of attempts.
- **Triage Agent** routes what's left: auto-resolve / explain / escalate. Every record visited exactly once, never loops.
- **Audit Agent** writes 20+ columns per decision to SQLite. Every rupee on any dashboard traces back here.
- **Q&A Agent** parses intent, picks the deterministic tool, computes, verifies, phrases. The verifier can reject the LLM's answer if any rupee doesn't ground; the deterministic answer is used instead.

---

## Why it's not just a matcher

**Bounded LLM.** The AI parses your question and writes the sentence. That's it. Every number in the sentence is computed in Python, then magnitude-checked by the verifier before you see it.

**Verifier veto.** If the LLM invents a figure (or gets one out by a paisa), the verifier catches it and swaps in the deterministic answer. Tested in `tests/test_verifier.py` with fabricated numbers, near-misses, and sign flips.

**Injection-safe.** The LLM never sees raw file text. It sees pre-computed numbers only. A file can't smuggle instructions.

**Full audit trail.** Every decision, every fuzzy score, every LLM call with its cost, latency, and verifier verdict, all in SQLite. `make eval` reads from it. So does the Dashboard. So does Ask.

---

## Metrics (from `make eval`)

| Metric | Value | Notes |
|---|---|---|
| Match rate (exact + fuzzy) | **97.68%** | 512 active settlements |
| Reconciled rate | **94.78%** | fully tied order↔settlement↔bank, fee-validated |
| Throughput (small batch) | **~12,700 rec/s** | 500 records in 0.40s |
| Stress benchmark | **149,250 rows in 217s** | 687 rec/s sustained |
| Exception detection F1 | **0.93** | TP 31 · FP 5 · FN 0 |
| False negatives | **0** | never misses real money |
| Q&A routing / grounding | **10/10 · 10/10** | on seed set |
| AI cost per explanation | **~$0.00007** | on-demand, capped, verifier-gated |
| Razorpay schema fidelity | **7/11 exact** | validated live against test API |

Recall pinned to 1.0 by choice: the five false positives are all sub-paise GST rounding a human clears in seconds. I'd rather flag five clean rows than miss one real overcharge. Documented in `config.py` with the exact reasoning.

---

## Screens

### Upload page

Three drop-zones for the three files, per-file validation, and a live agent panel that lights up cell by cell as each agent runs. Below the panel, a Decision Ledger streams every per-record verdict written to SQLite.

![Upload page with three drop zones](docs/img/upload.png)

### The agents in flight

Every agent has its own tile on the panel. Status flips from standby to running to verified as the run progresses. Hover any tile for a plain-English explanation of what that agent does and why.

![Agent panel showing all seven agents lit up mid-run](docs/img/agents.png)

### Dashboard

Three tabs. **Cash** carries the four-bucket Cash Position (Cleared / In-flight / At-risk / Ghost) with a Net Available headline, a dual-direction Cash Timeline (past landings + future T+2 projections), Tax Exposure (MDR / GST / TCS drift), and Fee Slab revenue advice. **Reconciliation** carries the breakdown, Repair Agent activity, fee composition, and the source-file viewer. **Audit** carries either Detection Accuracy (demo) or Quality Signals A/B/C/D (uploads) plus the AI cost strip. The Truth-Anchor Agent card sits above all tabs; it appeals every exception to Razorpay's live API and shows the drift when your CSV disagrees.

![Dashboard with Cash Position, Cash Timeline, Truth-Anchor card](docs/img/dashboard.png)

### Exceptions

Every record ReconMint could not confidently resolve lands here. Severity tabs, payment-id search, and a drawer with five tabs: **Overview**, **Diagnose & Fix** (a persistent checklist keyed to the exception category), **Ledger** (fee reconstruction), **Decisions** (the Repair Agent's per-record tree of attempts), and **Appeal to Razorpay** (live API tiebreaker). Resolving an exception emits an adjustment memo in JSON and printable HTML.

![Exceptions page with drawer open showing Repair Agent decision tree](docs/img/exceptions.png)

### Ask the agent

Plain-text question box. The trace shows every step: understand intent, choose tool, compute, verify, phrase. Every answer carries a green Verified tick when the hallucination verifier approved every rupee. Click Prove-it to expand the receipts, showing the exact payment IDs behind the number.

![Ask the agent page with verified answer and prove-it receipts](docs/img/asktheagent.png)

---

## The engine (money math)

- **Integer paise everywhere.** No floats near money.
- **IST-normalized dates.** Monday settlement + Wednesday bank credit still compares as T+2.
- **Independent fee reconstruction.** Every settlement's expected net is recomputed from `gross − MDR − GST-on-MDR − TCS − refund − chargeback`. If the bank credit matched but the net was wrong, it's still flagged.
- **Materiality threshold** on fee gaps: `max(a few paise, 0.05% of gross)`. Real overcharges are 0.4-1.2% of gross, so this doesn't hurt recall but stops sub-paise rounding artifacts flooding the exception list.

---

## Data + validation

`backend/generator/generate.py` emits `orders.csv`, `settlement.csv`, `bank.csv`, plus a hidden `answer_key.csv` with ground-truth categories. The matcher never sees the key; the eval harness scores against it.

The settlement schema is modelled field-for-field on the real Razorpay API, validated by `scripts/razorpay_probe.py` against live test-mode Orders. Test mode doesn't emit multi-day settlement files (needs live activation and real payout cycles), so ReconMint **generates** a batch whose **field shapes are real** while **volume and timing are synthetic and disclosed.** Full validation table in `docs/razorpay_validation.md`.

---

## Reproduce every number

```bash
python scripts/run_eval.py       # match rate + precision/recall/F1
python scripts/eval_qa.py        # Q&A agent routing + grounding
python scripts/benchmark.py      # throughput at 500 / 2k / 10k / 50k
python scripts/razorpay_probe.py # live schema check
make test                        # verifier + engine tests
```

---

## Repo layout

```
backend/
  agent/          the agents themselves + audit trail + LLM plumbing
  api/            FastAPI endpoints + Pydantic response models
  engine/         deterministic money math (loader, matcher, fuzzy, fee reconstruction)
  eval/           ground-truth harness (produces the F1 on the demo dataset)
  generator/      synthetic dataset generator + hidden answer key
frontend/         React + Vite, ledger-paper aesthetic
scripts/          stress benchmark, live Razorpay probe, eval runner
tests/            verifier + engine
docs/             Razorpay schema validation notes + architecture image
FAILURES.md       every bug that hit me during the build, in my own words
```

## The main backend files (if you're reading the code)

Start here if you're reviewing what the agents actually do. Every file listed is small and readable end-to-end.

- **`backend/agent/orchestrator.py`** — the loop that runs the whole reconciliation. Reads the three files, calls each agent in order, threads the record between them, and writes the audit trail. Every "stage" you see in the frontend cockpit is one function call from here.
- **`backend/agent/repair.py`** — the Repair Agent. Three strategy functions (`normalize_utr`, `widen_date_window`, `amount_utr_fuzzy`), the 85 percent confidence gate, the first-wins loop, and the amount-bucket + UTR indexes that make it near-linear. This is the file to open if you want to see what "an agent branching per record" looks like in code.
- **`backend/agent/qa.py`** — the Q&A agent. Parses intent (LLM + keyword fallback), picks a deterministic computation, runs it, hands the result to the verifier before phrasing the sentence. Never touches raw file bytes so there's nothing for a prompt injection to hide in.
- **`backend/agent/verifier.py`** — the hallucination verifier. Extracts every rupee figure the LLM wrote and magnitude-compares them against the set of computed values the deterministic layer produced. If any figure doesn't match, the LLM's phrasing is rejected and the deterministic answer is used instead. This is the guard rail.
- **`backend/agent/razorpay_client.py`** — the live Razorpay API client. Used at ingest to prove the pipeline is grounded, and per-exception by the Truth-Anchor Agent. Timeout-bounded, degrades cleanly if keys are missing.
- **`backend/agent/audit.py`** — the SQLite audit trail. Every decision, every attempt, every LLM call with its cost and verifier verdict, all in one table you can query.
- **`backend/api/main.py`** — the FastAPI layer. All endpoints live here. If you want to see what the frontend fetches, this is the file.
- **`backend/engine/loader.py`** — reads CSV or Excel, detects header rows through preambles, fuzzy-maps 60+ merchant-export column aliases to the canonical schema, coerces amounts to integer paise, normalizes dates to Asia/Kolkata.
- **`backend/engine/matcher.py`** — the exact-match pass. Independently reconstructs the fee schedule (MDR + GST-on-MDR + TCS) and joins on UTR + amount + IST day.

---

## Track 4 alignment

Reading the brief line by line, this is what I built for each phrase:

- *"Run the books."* Three-way reconciliation with integer paise, independent fee reconstruction, full SQLite audit.
- *"And the cash position."* Four-bucket Cash Position card with a Net Available headline, sourced from this run's audited decisions.
- *"AI agent that closes one finance-ops loop."* Resolve an exception, get back an adjustment memo (JSON + printable HTML) with a simulated ERP webhook payload. That's the loop closing.
- *"50+ record batches."* 149,250 rows in 217s on the stress benchmark. 500 records in 0.4s.
- *"Match rate."* Live headline strip, mutually-exclusive stacked breakdown, per-strategy Repair Agent hit rates.
- *"Exceptions it could not resolve."* Full Exceptions table, nothing hidden, resolution reasons and notes persisted per row.
- *"Verification, not generation."* Hallucination verifier magnitude-checks every rupee an LLM writes. Rejects → deterministic answer used.

Four directions the brief suggested, all shipped in the same agent:

- **Multi-source reconciliation** → Match + Fuzzy + Repair (three strategies per unmatched record).
- **Settlement Q&A agent** → Ask page with the parse → compute → verify → phrase loop.
- **Forward cash forecaster** → dual-direction Cash Timeline, T+2 projection, past-due surfacing.
- **Tax-line matcher** → MDR / GST / TCS observed vs expected with recover / reserve exposure.

---

## A note on the build

I keep [FAILURES.md](FAILURES.md) open in a tab while I work. Every real bug goes in it the day it breaks. The float-rupee drift on day one. The verifier that was too strict and rejected correct explanations on day seven. The O(n²) fuzzy pass I only caught with a stress benchmark. The Repair Agent that scaled cubically until I built the amount + normalized-UTR indexes. The Railway 500 from `requests` missing in `requirements.txt`. Each entry: symptom, root cause, fix, lesson.

I'd previously built Polynous, a multi-agent research system. The agent scaffolding here wasn't accidental; it's how I default to thinking about problems where small components need to disagree and vote.
