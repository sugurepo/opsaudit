# OpsAudit — Final Project Plan

**Document-expiry triage pipeline: AI reads the documents, plain code decides, a human reviews the doubtful ones.**

Plan version: November 24, 2026. Prepared for Joannah Jean Dorono. Target: AI / data / automation junior roles (for example Woven by Toyota data and automation postings).

---

## 0. How to use this file in a new conversation

Paste this whole file as your first message, then send this opening message after it:

> This is my project plan. I'm a junior developer. I know C#, JavaScript, React, SQL basics and Google Apps Script, and I use Claude Code and Copilot. I am new to Python, FastAPI, Pytest and LLM APIs. Please guide me through ONE milestone at a time, starting with Milestone 0. Rules: explain every concept in plain words before we write code; after each module ask me to explain it back in 3 sentences; give me commands for [Windows PowerShell / macOS / Linux, tell me which you use]; never invent results, numbers or features; and do not touch my resume until the Definition of Done in section 17 is fully ticked.

**Rules for whoever helps build this (human or AI):**

1. The owner must be able to explain every file in an interview. Prefer simple, readable code over clever code.
2. Build in small vertical slices. Run it after every slice.
3. Synthetic data only. Never upload a real document to any AI service (see section 3).
4. Never claim on a resume or README something that isn't built and measured.
5. Facts marked **[verify]** came from web sources on Nov 24, 2026 and may have changed. Check the official page on Day 0.

### Plain-language glossary

| Term | Meaning |
|---|---|
| Pipeline | A fixed series of steps every document goes through |
| Schema | The exact shape of data (field names and types) |
| Deterministic | Same input always gives the same output (plain code is; AI is not) |
| Idempotent | Running the same step twice gives the same result and doesn't create duplicates |
| Ground truth | The correct answers you wrote down yourself, used to grade the AI |
| Mock / fake | A stand-in for the real AI in tests, so tests are fast, free and need no key |
| CI | A robot (GitHub Actions) that runs your tests every time you push code |
| Hash | A fingerprint of a file's bytes; identical files give identical hashes |
| Quota / 429 | The AI service refusing requests because you used too many too fast |

---

## 1. Goal, scope and success

### The problem it models
Staff submit certificates, training records and vendor compliance documents. Someone has to open each one, read the dates, spot missing fields or signatures, track upcoming expiries and chase people. It is slow and error-prone. This mirrors the pilot records automation you built at INAEC, upgraded from fixed rules to an AI-assisted pipeline.

### What it does (MVP)
1. Take a batch of synthetic PDFs and images.
2. Skip duplicates (file fingerprint).
3. Ask an AI model to read each one into a fixed structure (name, ID, issue date, expiry date, issuer, signature present).
4. Check the AI's answer with plain code (valid dates? expiry after issue? required fields present?).
5. Apply compliance rules in plain code: VALID, EXPIRING_SOON, EXPIRED, or NEEDS_REVIEW.
6. Save everything in SQLite, including the raw AI output and an append-only audit log.
7. Let a human review the NEEDS_REVIEW queue through a small web page.
8. Measure how accurate it is against your own answer sheet.

### Non-goals (do not build these)
Real documents, user login, multi-user support, cloud deployment, OCR engines, PostgreSQL, async code, a polished design.

### Stretch (only if on schedule, in this order)
1. A regex-only baseline extractor to compare against the AI (shows whether the AI adds value).
2. A second model in the evaluation table.
3. React version of the review page.
4. Google Sheets export.
5. Dockerfile.

### Success criteria
Section 17 (Definition of Done).

---

## 2. Decisions locked, with reasons

| Decision | Choice | Why |
|---|---|---|
| Language | Python 3.12 or newer | Required by the target roles; wide library support |
| API framework | FastAPI | Small, readable, auto docs at `/docs` |
| Database | SQLite via the standard `sqlite3` module and plain `.sql` files | No server, no ORM magic; you can explain every table and JOIN |
| AI extraction | Gemini via the official `google-genai` SDK, behind an adapter | Reads PDFs/images natively; free tier exists; adapter makes the model replaceable |
| AI's job | Read messy documents into fields. Nothing else | The AI never decides expiry. Plain code does |
| Review UI | One static HTML page served by FastAPI (vanilla JS) | You already show React elsewhere; saves 4+ hours |
| Tests | Pytest with a fake AI client | Fast, free, no key in CI |
| Lint/format | Ruff | One tool for both |
| CI | GitHub Actions | Runs ruff and pytest on every push |
| Config | `config.yaml`, `rules.yaml`, `.env` for the key | Nothing hardcoded |
| Packaging | Flat package `opsaudit/`, venv + pip | Least tooling to learn |

### Improvements over the earlier blueprint (and why)
- **Missing fields are computed by code**, not reported by the AI. A field is missing if it's null. Fewer places for the AI to be wrong.
- **EXPIRED is its own status**, not buried in NEEDS_REVIEW. A confidently read expired document is a finding, not a doubt.
- **The AI returns dates as text; our code parses them.** That lets us catch `2026-02-30` ourselves.
- **Model "confidence" is a weak hint only.** The hard checks are in code.
- **Response cache.** Every AI answer is saved to disk keyed by file hash + model + prompt version, so re-running evaluations doesn't burn quota. This also makes the project resilient to the free-tier limits in section 3.
- **Dev/holdout split** in the ground truth, so prompt tuning doesn't inflate the final numbers.
- **Append-only audit log enforced by database triggers**, so the claim is true inside the app.
- **Reference date is a parameter**, so tests and evaluation give the same answer every day.

---

## 3. Verified facts and open risks about the AI service (Nov 24, 2026) **[verify]**

These come from Google's docs pages and third-party articles that disagreed in places. The Day-0 probe (Milestone 0) settles each point on your own key.

1. **Model choice.** Google's deprecations page says gemini-2.0-flash was scheduled to shut down on June 1, 2026, and that gemini-2.5-flash is not deprecated but, for new projects, to use 3.5 Flash-Lite or 3.8 Flash. It also says access to 2.5 models is limited to users who have actively used them. Do not hardcode a model name. Read it from `config.yaml`.
2. **Free-tier quotas differ hugely by model.** A third-party snapshot lists about 20 requests/day for 3.8 Flash and 3.6 Flash, and about 500/day for 3.5 Flash-Lite at about 10 requests/minute. Another writer measured a free key where 3.8 Flash returned "limit: 0" (the model has no free quota on that account) while 3.5 Flash-Lite worked. Limits are per project (not per key) and daily quotas reset at midnight Pacific time. That is about 4:00 PM in Manila from November to March, and 3:00 PM from March to November.
3. **Best default model for this project: the current Flash-Lite model** (for example `gemini-3.5-flash-lite`), because its daily quota is large enough for 30 documents × several experiments. Confirm on Day 0.
4. **Two API styles exist.** The newer Interactions API (`client.interactions.create`, SDK 2.x) and the older `generate_content` with `response_schema`. Google's docs pages disagree on whether Interactions is beta or generally available, and one page says Interactions is now recommended. Both can return JSON matching a Pydantic schema. Design: one adapter file (`extractor/gemini.py`) hides the style. Switching is a one-file change.
5. **Always validate the AI's JSON yourself** with Pydantic (`Model.model_validate_json(text)`). A known SDK issue returns `None` for parsed output when the response hits the output-token limit, so set a generous `max_output_tokens` (at least 1024; thinking models may count thinking tokens too).
6. **Temperature.** One integration doc warns that changing temperature on Gemini 3+ models can degrade results. Leave temperature unset (model default) unless current docs say otherwise.
7. **Privacy.** Free-tier prompts and files can be used by Google to improve its products. This is why all documents are synthetic. Never upload INAEC or any real document.
8. **Cost if you ever need paid.** Flash-Lite input pricing is about $0.30 per million tokens **[verify]**. 100 test calls would cost pennies. Free is expected to be enough.
9. **Fallback if the free tier gives you "limit: 0" or no access:** (a) try the other Flash-Lite model, (b) enable billing on the project for pennies, (c) build the regex baseline extractor first and add the AI after.

---

## 4. Tech stack and setup

### Python packages
Runtime: `pydantic>=2`, `fastapi`, `uvicorn`, `python-multipart` (file upload), `pyyaml`, `python-dotenv`, `google-genai` (use 2.x if you use Interactions), `httpx` (needed by FastAPI's test client).
Synthetic data: `reportlab`, `pillow`, `faker`.
Dev: `pytest`, `ruff`.
After installing, run `pip freeze > requirements.txt` equivalents (keep runtime and dev lists separate: `requirements.txt`, `requirements-dev.txt`) so versions are recorded.

### Environment setup (Milestone 0)
```
python -m venv .venv
# Windows PowerShell:   .venv\Scripts\Activate.ps1
# macOS / Linux:        source .venv/bin/activate
pip install -r requirements.txt -r requirements-dev.txt
```
Create `.env` (never committed) with `GEMINI_API_KEY=...`. Create `.env.example` (committed) with a blank value. Get the key from Google AI Studio.

### Repository layout
```
opsaudit/                       <- repo root
  README.md
  LICENSE                       (MIT)
  .gitignore                    (.env, .venv, data/db, data/cache, __pycache__)
  .env.example
  config.yaml
  rules.yaml
  pyproject.toml                (ruff + pytest settings only)
  requirements.txt
  requirements-dev.txt
  .github/workflows/ci.yml
  prompts/v1.txt
  prompts/v2.txt
  sql/schema.sql
  sql/queries/                  (saved reports, one .sql per question)
  data/synthetic/               (committed: 30 fake docs + ground_truth.csv)
  data/inbox/                   (gitignored: where batches are dropped)
  data/cache/                   (gitignored: saved AI answers)
  data/db/                      (gitignored: SQLite files)
  scripts/generate_synthetic.py
  scripts/probe_model.py
  docs/architecture.md          (Mermaid diagram)
  docs/eval_results.md          (committed results + failure analysis)
  opsaudit/
    __init__.py
    __main__.py                 (python -m opsaudit ...)
    cli.py
    config.py
    models.py                   (Pydantic models)
    db.py
    hashing.py
    extractor/base.py           (interface + result type)
    extractor/gemini.py         (real AI + throttle + retry + cache)
    extractor/fake.py           (tests)
    validator.py                (parse and sanity-check AI output)
    rules.py                    (pure function: record + rules + date -> verdict)
    pipeline.py                 (ingest, extract, validate, rules, store)
    evaluate.py                 (metrics against ground truth)
    reports.py
    api/app.py                  (FastAPI routes)
    api/static/review.html
  tests/
    conftest.py
    test_rules.py
    test_validator.py
    test_pipeline.py
    test_db.py
    test_api.py
    test_config.py
    test_evaluate.py
```

---

## 5. Data model

### Pydantic model (what the AI must return)
```python
from typing import Literal, Optional
from pydantic import BaseModel, Field

DocType = Literal["medical_certificate", "training_certificate", "vendor_compliance", "unknown"]

class ExtractedRecord(BaseModel):
    document_type: DocType
    subject_name: Optional[str] = None
    certificate_id: Optional[str] = None
    issue_date: Optional[str] = Field(None, description="YYYY-MM-DD or null")
    expiry_date: Optional[str] = Field(None, description="YYYY-MM-DD or null")
    issuer: Optional[str] = None
    signature_present: Optional[bool] = None
    model_confidence: Optional[float] = Field(None, ge=0, le=1)
```
If the AI service handles nullable fields badly, the Day-0 probe will show it. Fallback: ask for empty strings and convert `""` to `None` in the validator.

### Statuses
| Status | Meaning |
|---|---|
| VALID | All checks pass and days remaining is more than the threshold |
| EXPIRING_SOON | Days remaining is between 0 and the threshold (inclusive). Expiring today counts here |
| EXPIRED | Days remaining is below 0 |
| NEEDS_REVIEW | Any validation problem. Dates can't be trusted, so no date verdict is given |

Review queue = NEEDS_REVIEW only. EXPIRED and EXPIRING_SOON are alerts reported to users.

### Reason codes (stored with every NEEDS_REVIEW)
`UNKNOWN_DOC_TYPE`, `MISSING_FIELD:<name>`, `INVALID_DATE:<field>`, `EXPIRY_BEFORE_ISSUE`, `IMPLAUSIBLE_VALIDITY`, `ISSUE_DATE_IN_FUTURE`, `SIGNATURE_MISSING`, `LOW_CONFIDENCE`, `EXTRACTION_FAILED`, `SCHEMA_INVALID`.

### SQLite schema (`sql/schema.sql`)
```sql
PRAGMA foreign_keys = ON;

CREATE TABLE IF NOT EXISTS documents (
  id           INTEGER PRIMARY KEY,
  file_hash    TEXT NOT NULL UNIQUE,      -- SHA-256 of file bytes (idempotency)
  file_name    TEXT NOT NULL,
  file_path    TEXT NOT NULL,
  size_bytes   INTEGER NOT NULL,
  ingested_at  TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS pipeline_runs (
  id                 INTEGER PRIMARY KEY,
  started_at         TEXT NOT NULL,
  finished_at        TEXT,
  model_name         TEXT,
  prompt_version     TEXT,
  as_of_date         TEXT NOT NULL,
  docs_seen          INTEGER NOT NULL DEFAULT 0,
  docs_processed     INTEGER NOT NULL DEFAULT 0,
  docs_failed        INTEGER NOT NULL DEFAULT 0,
  duplicates_skipped INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE IF NOT EXISTS extractions (
  id            INTEGER PRIMARY KEY,
  document_id   INTEGER NOT NULL REFERENCES documents(id),
  run_id        INTEGER REFERENCES pipeline_runs(id),
  source        TEXT NOT NULL CHECK (source IN ('model','human')),
  model_name    TEXT,
  prompt_version TEXT,
  raw_response  TEXT,                      -- exactly what the AI returned
  parsed_json   TEXT,                      -- validated record as JSON, or NULL
  schema_valid  INTEGER NOT NULL,          -- 1 or 0
  error         TEXT,
  latency_ms    INTEGER,
  input_tokens  INTEGER,
  output_tokens INTEGER,
  from_cache    INTEGER NOT NULL DEFAULT 0,
  created_at    TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS verdicts (
  id             INTEGER PRIMARY KEY,
  document_id    INTEGER NOT NULL REFERENCES documents(id),
  extraction_id  INTEGER NOT NULL UNIQUE REFERENCES extractions(id),
  status         TEXT NOT NULL CHECK (status IN ('VALID','EXPIRING_SOON','EXPIRED','NEEDS_REVIEW')),
  days_remaining INTEGER,
  reasons        TEXT NOT NULL DEFAULT '[]',   -- JSON array of reason codes
  as_of_date     TEXT NOT NULL,
  created_at     TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS review_actions (
  id             INTEGER PRIMARY KEY,
  verdict_id     INTEGER NOT NULL REFERENCES verdicts(id),
  action         TEXT NOT NULL CHECK (action IN ('CONFIRMED','CORRECTED','REJECTED')),
  reviewer       TEXT NOT NULL,
  corrected_json TEXT,
  note           TEXT,
  created_at     TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS audit_log (
  id          INTEGER PRIMARY KEY,
  event_type  TEXT NOT NULL,
  entity_type TEXT NOT NULL,
  entity_id   INTEGER,
  detail      TEXT,
  created_at  TEXT NOT NULL
);

CREATE TRIGGER IF NOT EXISTS audit_log_no_update
BEFORE UPDATE ON audit_log
BEGIN SELECT RAISE(ABORT, 'audit_log is append-only'); END;

CREATE TRIGGER IF NOT EXISTS audit_log_no_delete
BEFORE DELETE ON audit_log
BEGIN SELECT RAISE(ABORT, 'audit_log is append-only'); END;

-- "Current" verdict = the most recent verdict for each document
CREATE VIEW IF NOT EXISTS current_verdicts AS
SELECT v.*
FROM verdicts v
JOIN (SELECT document_id, MAX(id) AS max_id FROM verdicts GROUP BY document_id) m
  ON v.id = m.max_id;

CREATE INDEX IF NOT EXISTS idx_verdicts_status ON verdicts(status);
CREATE INDEX IF NOT EXISTS idx_extractions_doc ON extractions(document_id);
```

### Saved queries (`sql/queries/`)
```sql
-- expiring_soon.sql
SELECT d.file_name, cv.days_remaining
FROM current_verdicts cv JOIN documents d ON d.id = cv.document_id
WHERE cv.status = 'EXPIRING_SOON'
ORDER BY cv.days_remaining;

-- status_counts.sql
SELECT status, COUNT(*) AS n FROM current_verdicts GROUP BY status ORDER BY n DESC;

-- review_queue.sql
SELECT cv.id AS verdict_id, d.file_name, cv.reasons
FROM current_verdicts cv
JOIN documents d ON d.id = cv.document_id
LEFT JOIN review_actions ra ON ra.verdict_id = cv.id
WHERE cv.status = 'NEEDS_REVIEW' AND ra.id IS NULL;

-- model_prompt_stats.sql
SELECT model_name, prompt_version,
       COUNT(*) AS calls,
       ROUND(AVG(latency_ms)) AS avg_latency_ms,
       SUM(input_tokens) AS input_tokens,
       SUM(output_tokens) AS output_tokens,
       SUM(schema_valid) AS schema_valid_count
FROM extractions WHERE source = 'model'
GROUP BY model_name, prompt_version;

-- reviewer_override_rate.sql
SELECT action, COUNT(*) AS n FROM review_actions GROUP BY action;
```

---

## 6. Configuration

### `config.yaml`
```yaml
ai:
  model_name: "SET_ON_DAY_0"        # from the probe; never hardcode elsewhere
  api_style: "interactions"          # or "generate_content"; decided on Day 0
  prompt_version: "v2"
  max_output_tokens: 1024
  max_retries: 3
  min_seconds_between_calls: 7       # stays under ~10 requests/minute
  use_cache: true
  cache_dir: "data/cache"
  # temperature intentionally not set (see section 3)
review:
  low_confidence_threshold: 0.7
paths:
  inbox: "data/inbox"
  db: "data/db/opsaudit.db"
  prompts_dir: "prompts"
```

### `rules.yaml`
```yaml
required_fields: [subject_name, certificate_id, issue_date, expiry_date, issuer]
document_types:
  medical_certificate:
    expiring_soon_days: 60
    max_validity_years: 5
    requires_signature: true
  training_certificate:
    expiring_soon_days: 30
    max_validity_years: 5
    requires_signature: false
  vendor_compliance:
    expiring_soon_days: 30
    max_validity_years: 3
    requires_signature: true
```
The thresholds are illustrative (this is a portfolio project, not real aviation regulation). Say so in the README.

---

## 7. Pipeline design

```
ingest -> pre-check -> extract -> validate -> rules -> store -> report
```

| Stage | What happens | Failure behavior |
|---|---|---|
| Ingest | Accept `.pdf`, `.png`, `.jpg`. Reject other types and files over 10 MB | Logged in audit_log, batch continues |
| Pre-check | Compute SHA-256. If it's already in `documents`, skip (duplicate) | Counted in `duplicates_skipped` |
| Extract | Look in the cache first. If missing, call the AI (throttled, with retries). Save raw response | Failure stored as an extraction with `error`. Verdict becomes NEEDS_REVIEW with EXTRACTION_FAILED |
| Validate | Parse JSON into `ExtractedRecord`. Parse dates strictly as `YYYY-MM-DD` | Bad JSON gives SCHEMA_INVALID. Bad date gives INVALID_DATE |
| Rules | Pure function decides the status | No I/O, no failure modes |
| Store | One transaction writes extraction, verdict, audit rows | Rolled back together if anything fails |
| Report | SQL queries, CSV export | n/a |

Properties to guarantee (and test):
- **One bad file never stops the batch.**
- **Re-running the same inbox creates no duplicate rows.**
- **Raw AI output is always stored**, even when invalid, so failures can be debugged.
- **The reference date ("as of") is passed in**, never read inside the rules.

### Rules engine (`rules.py`), pure function
Order of checks (collect all reasons, then decide):
1. Extraction missing or schema invalid gives NEEDS_REVIEW and stops.
2. `document_type == "unknown"` gives UNKNOWN_DOC_TYPE and stops.
3. Each required field that is null or blank gives `MISSING_FIELD:<name>`.
4. Parse `issue_date` and `expiry_date` with `datetime.strptime(s, "%Y-%m-%d").date()`. A failure (including `2026-02-30`) gives `INVALID_DATE:<field>`.
5. If both parsed: expiry before issue gives EXPIRY_BEFORE_ISSUE. Validity longer than `max_validity_years * 366` days gives IMPLAUSIBLE_VALIDITY. Issue date after `as_of` gives ISSUE_DATE_IN_FUTURE.
6. If `requires_signature` and `signature_present` is not True, add SIGNATURE_MISSING.
7. If `model_confidence` is below the threshold, add LOW_CONFIDENCE (skipped when the record came from a human).
8. If any reasons: NEEDS_REVIEW.
9. Otherwise `days_remaining = (expiry - as_of).days`: below 0 gives EXPIRED, 0 to threshold gives EXPIRING_SOON, above threshold gives VALID.

Boundary rule, documented in the README: a certificate is valid through its expiry date (inclusive).

### AI adapter (`extractor/`)
```python
class ExtractionResult(BaseModel):
    raw_text: Optional[str]
    input_tokens: Optional[int]
    output_tokens: Optional[int]
    latency_ms: Optional[int]
    from_cache: bool = False
    error: Optional[str] = None

class Extractor(Protocol):
    def extract(self, file_path: Path) -> ExtractionResult: ...
```
`pipeline.py` receives an `Extractor` as a parameter (dependency injection). Tests pass `FakeExtractor`.

`GeminiExtractor` responsibilities:
1. **Cache lookup** by `{sha256}_{model}_{prompt_version}.json`.
2. **Throttle**: at least `min_seconds_between_calls` between real calls.
3. **Call the model** with the prompt text plus the file. Copy the exact call pattern (including how a PDF or image is attached) from Google's current structured-output docs on Day 0. Ask for JSON matching `ExtractedRecord.model_json_schema()`.
4. **Retry** on 5xx and per-minute 429 with exponential backoff (about 5 s, 10 s, 20 s), up to `max_retries`.
5. **Fail fast on a daily-quota error**: detect it by looking for "PerDay" in the error text **[verify]**. Raise a clear message ("daily quota used up; resume after midnight Pacific, or use the cache").
6. **One retry for malformed JSON**, then record SCHEMA_INVALID.
7. **Record** latency, tokens, model, prompt version.
8. Never log full document contents.

### Prompts
`prompts/v1.txt` (deliberately minimal):
```
Extract the requested fields from this document and return JSON.
```

`prompts/v2.txt`:
```
You extract structured data from one compliance document (PDF or image).
Return only JSON matching the provided schema. No commentary.

Rules:
- document_type: medical_certificate (a medical fitness certificate for a person),
  training_certificate (a course or safety training completion certificate for a person),
  vendor_compliance (an insurance or compliance certificate for a company),
  unknown (anything else, including blank or unrelated pages).
- subject_name: the person or company the certificate was issued to.
  Not the issuer and not a signatory.
- certificate_id: the certificate, reference or licence number, exactly as printed.
- issue_date: the date the certificate was issued, as YYYY-MM-DD.
  Do not use a date of birth, print date or signature date unless it is labelled as the issue date.
- expiry_date: the date it expires or is valid until, as YYYY-MM-DD.
- If a date is missing, unreadable or ambiguous (for example 04/05/2027 could be
  day-first or month-first), return null. Never guess.
- issuer: the organisation that issued it.
- signature_present: true only if a signature or signature stamp is visible, otherwise false.
- model_confidence: 0.0 to 1.0, how sure you are that every field is correct and legible.
- If a field is not on the document, return null. Do not infer values.
```
The prompts contain no business rules. The AI never decides what is expired.

---

## 8. Synthetic data and ground truth

### Rules for synthetic data
- Every document carries a footer: **"SYNTHETIC SAMPLE. NOT A REAL DOCUMENT."**
- Names from `Faker` with a fixed seed. Issuers are obviously fictional (for example "Example Aviation Medical Board").
- Generation is reproducible: `python scripts/generate_synthetic.py --seed 42`.
- PDFs are made with `reportlab`. Image scans are drawn directly with Pillow (`ImageDraw`) and then degraded (blur, noise, 2° rotation, lower resolution). This avoids needing Poppler to convert PDFs to images.
- Use `ImageFont.load_default(size=...)` or a bundled font **[verify the Pillow version supports the size argument]**.
- All dates are generated relative to a fixed reference date: **`EVAL_AS_OF = 2026-12-01`**. The evaluation always runs with `--as-of 2026-12-01`, so results never change with the calendar.

### The 30 documents (scenario table)
| Scenario | Count | Notes |
|---|---|---|
| Clean valid | 7 | Mix of the three document types |
| Expiring soon | 6 | Include boundaries: expires exactly on the as-of date, exactly at the threshold, one day over the threshold (that one is VALID) |
| Expired | 6 | 1 day, 30 days, 400 days ago |
| Missing field | 3 | No expiry date; no certificate ID; no issuer |
| Signature missing | 2 | Document types that require signatures |
| Date logic error | 2 | Expiry before issue; 12-year validity |
| Ambiguous date | 1 | Numeric `04/05/2027`. Expected outcome: AI returns null, so NEEDS_REVIEW |
| Unknown type | 1 | A page that isn't a certificate |
| Distractor dates | 2 | Date of birth and "printed on" dates present. The AI must not confuse them |
| **Total** | **30** | |

Overlays: render **4** of the 30 as degraded PNG scans. Use at least 3 different date formats across the set (`2027-03-05`, `05 Mar 2027`, `March 5, 2027`).
Extra files **not** counted in the 30: **2 byte-identical duplicate copies** to test duplicate rejection.

### `data/synthetic/ground_truth.csv` columns
`file_name, split, scenario, document_type, subject_name, certificate_id, issue_date, expiry_date, issuer, signature_present, expected_status, expected_reasons`

- `split`: **20 dev, 10 holdout**, stratified so each scenario appears in both where possible.
- **Write `expected_status` by hand from the scenario, not by calling your rules code.** Otherwise you are grading the code with itself.
- Blank cells mean the correct answer is null.

---

## 9. Evaluation

### Command
```
python -m opsaudit eval --prompt v2 --split dev --as-of 2026-12-01
python -m opsaudit eval --prompt v2 --split holdout --as-of 2026-12-01
```
Each experiment uses its own temporary database, so runs don't mix.

### Methodology rule
Tune prompts only on **dev**. Run **holdout once per final prompt**. Do not edit prompts after seeing holdout results. Report both numbers.

### Metrics (all reported as counts and percentages)
| Metric | Definition |
|---|---|
| Schema-valid rate | Responses that parse into `ExtractedRecord` / total |
| Field accuracy (per field) | Exact match after trimming and case-folding names; dates must match exactly; null equals null |
| Document exact match | Documents where every field is correct / total |
| Status accuracy | Predicted status equals expected status / total |
| Review-queue recall | Expected NEEDS_REVIEW that were caught / expected NEEDS_REVIEW |
| Review-queue precision | Predicted NEEDS_REVIEW that were truly NEEDS_REVIEW / predicted NEEDS_REVIEW |
| **False clears** | Expected EXPIRED, EXPIRING_SOON or NEEDS_REVIEW but predicted VALID. This is the dangerous error. Report the count and list every case |
| Latency | Mean and 95th percentile (milliseconds) |
| Tokens and cost | Total input/output tokens; $0 on the free tier |

With 30 documents, one document is about 3 percentage points. Always show counts such as "23/25", not percentages alone.

### Experiment matrix
| Run | Model | Prompt | Calls |
|---|---|---|---|
| E1 | Flash-Lite (config) | v1 | 30 |
| E2 | Flash-Lite (config) | v2 | 30 |
| E3 (optional) | Second model | v2 | 30 |
| Optional baseline | none (regex) | n/a | 0 |

Time per run with 7 s spacing: about 3.5 minutes. The cache makes reruns free.

### `docs/eval_results.md` must contain
Results table, a **failure analysis** (what failed, why, what you changed), and a **limitations** section: synthetic documents are cleaner than real ones, 30 documents is a small sample, one model family, results are a snapshot of one date and prompt.

---

## 10. Testing plan

Rule: **CI never calls the real AI and never needs a key.** One optional test marked `@pytest.mark.live` is skipped unless you run `pytest -m live`.

Fixtures (`conftest.py`): in-memory SQLite with `schema.sql` loaded; `FakeExtractor` with canned responses and a call counter; fixed `as_of = date(2026, 12, 1)`; temp inbox with sample files.

| File | Tests |
|---|---|
| `test_rules.py` | valid; days = threshold gives EXPIRING_SOON; days = threshold + 1 gives VALID; expires today; expired yesterday; each required field missing gives NEEDS_REVIEW with the right code; `2026-02-30` gives INVALID_DATE; expiry before issue; implausible validity; issue in future; signature missing when required; signature not required; low confidence; unknown type; extraction failed; **validation failure beats expired** (precedence) |
| `test_validator.py` | accepts ISO dates only; rejects `05/03/2027` and `5 Mar 2027`; handles `null`/empty strings; invalid JSON gives SCHEMA_INVALID |
| `test_pipeline.py` | duplicate file skipped; re-running the inbox creates no new rows; one failed extraction doesn't stop the batch; malformed AI JSON gives NEEDS_REVIEW; simulated timeout retries then fails cleanly; cache hit means the extractor is not called; unsupported file type rejected |
| `test_db.py` | `UPDATE` and `DELETE` on `audit_log` raise; status CHECK constraint rejects bad values; `current_verdicts` returns the latest verdict only |
| `test_api.py` | upload processes a file; queue lists NEEDS_REVIEW items; review action writes `review_actions` and `audit_log`; correcting a record creates a human extraction and a new verdict |
| `test_config.py` | loads valid config; placeholder model name raises a clear error |
| `test_evaluate.py` | metrics on a tiny hand-made fixture equal hand-computed numbers (including a false clear) |

Target: about 30 tests. Name them as sentences so a test list reads like a spec. Keep a short `docs/test_cases.md` summarizing categories (this is honest evidence for "test case writing").

### CI (`.github/workflows/ci.yml`)
```yaml
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: ruff check .
      - run: ruff format --check .
      - run: pytest -q
```
**[verify]** newer major versions of `checkout` and `setup-python` may exist. Check their README pages. Add the CI status badge to the README.

---

## 11. API and review page

| Method and path | Purpose |
|---|---|
| `GET /health` | Liveness check |
| `POST /documents` | Upload one file, process it, return its record and verdict (or "duplicate") |
| `GET /documents?status=` | List documents with their current verdict |
| `GET /documents/{id}` | Full detail: record, verdict, reasons, extraction metadata |
| `GET /documents/{id}/file` | Serve the original file for the review page |
| `GET /review/queue` | NEEDS_REVIEW verdicts with no review action yet |
| `POST /review/{verdict_id}` | Body: `action`, `reviewer`, final field values, `note` |
| `GET /reports/summary` | Status counts |
| `GET /reports/export.csv` | CSV export of current verdicts |

**Review logic:** the reviewer sees fields prefilled from the AI and submits the final values.
- Same values: action `CONFIRMED`. Changed values: `CORRECTED`.
- The backend writes a **human extraction** (`source='human'`, confidence 1.0), re-runs the rules (skipping the LOW_CONFIDENCE check), stores the new verdict, and logs an audit event.
- `REJECTED` marks the document unusable and removes it from the queue.
- Everything is append-only, so history is preserved.

**Review page (`static/review.html`):** queue list on the left. Selecting an item shows the original (`<iframe>` for PDFs, `<img>` for images) next to a form with the fields. Buttons: Submit and Reject. Plain HTML and `fetch`. No build step. No styling beyond readable.

---

## 12. Command-line interface

```
python -m opsaudit init-db
python -m opsaudit process --inbox data/inbox [--as-of YYYY-MM-DD]
python -m opsaudit report [--query status_counts]
python -m opsaudit eval --prompt v2 --split dev --as-of 2026-12-01
python -m opsaudit serve
```
Use the standard `argparse` module. Scheduling is documented in the README as an optional cron / Task Scheduler line, not implemented in code.

---

## 13. Milestones, tasks and hours

Assumes 10 to 14 hours per week. Total realistic estimate: **32 to 42 hours** (the earlier 24 to 35 estimate did not include the cache, holdout split, UI and eval write-up). Minimum viable version (see cut order): about 24 to 28 hours.

| Milestone | Hours | Weeks (at about 2 h/day) |
|---|---|---|
| M0 Setup and Day-0 probe | 2 to 3 | Week 1 |
| M1 Synthetic data and ground truth | 4 to 5 | Week 1 |
| M2a Rules, validator, DB (no AI needed) | 5 to 6 | Week 1 to 2 |
| M2b AI adapter and pipeline | 6 to 8 | Week 2 |
| M3 Evaluation | 4 to 5 | Week 2 to 3 |
| M4 Tests and CI finish | 3 to 4 | Week 3 |
| M5 API and review page | 5 to 7 | Week 3 to 4 |
| M6 README, docs, polish | 3 to 4 | Week 4 |
| **Total** | **32 to 42** | **3 to 4 weeks** |

### M0 Setup and Day-0 probe (2 to 3 h)
- [ ] Create a GitHub repo `opsaudit` (public from the start, no secrets), add MIT license and `.gitignore` with `.env`, `.venv`, `data/db`, `data/cache`.
- [ ] Create venv, install packages, create `.env` with your key.
- [ ] Write `scripts/probe_model.py`: list models available to your key; make one tiny call with a small test PDF asking for JSON matching a 3-field Pydantic model; print the raw response, token usage, and any error text.
- [ ] Settle: which model works on your key (try Flash-Lite first), which API style works, whether nullable fields work, whether a PDF can be attached the way the docs show, and what your real quota is (look at the error text if you hit a limit).
- [ ] Write the results into `config.yaml` (`model_name`, `api_style`).
- [ ] Add a trivial test and the CI workflow. Push. **Done when:** CI is green and the probe prints valid JSON.
- [ ] Open GitHub Issues for any problems you hit and close them as you fix them (this is genuine bug tracking you can later describe).

### M1 Synthetic data and ground truth (4 to 5 h)
- [ ] Write `scripts/generate_synthetic.py` (seeded). Produce 30 documents per section 8, plus 2 duplicates.
- [ ] Write `ground_truth.csv` by hand from the scenario table, with the 20/10 split.
- [ ] Open at least 5 files yourself and confirm they look right. Compare a few against the CSV.
- **Done when:** the script regenerates identical files from the seed and the CSV has 30 rows.

### M2a Rules, validator, database (5 to 6 h)
- [ ] `models.py`, `config.py`, `db.py` (applies `schema.sql`), `hashing.py`.
- [ ] `validator.py` and `rules.py` as pure functions.
- [ ] Write `test_rules.py` and `test_validator.py` **as you go**.
- **Done when:** all rules tests pass with no network access.
- **Understanding check:** explain in 3 sentences why the rules engine takes `as_of` as a parameter.

### M2b AI adapter and pipeline (6 to 8 h)
- [ ] `extractor/base.py`, `fake.py`, then `gemini.py` (cache, throttle, retry, quota handling).
- [ ] `pipeline.py` and `cli.py` (`init-db`, `process`).
- [ ] First run on **one** document. Then 5. Then all 30. Watch quota.
- [ ] `test_pipeline.py` and `test_db.py`.
- **Done when:** `process` handles all 30 plus the 2 duplicates, the duplicates are skipped, and a second run changes nothing.
- **Understanding check:** explain why raw AI output is stored separately from validated fields.

### M3 Evaluation (4 to 5 h)
- [ ] `evaluate.py` and the `eval` command. `test_evaluate.py`.
- [ ] Run E1 (v1, dev). Read every failure. Write v2 improvements only from dev failures. Run E2 (v2, dev).
- [ ] Run holdout once for v1 and once for v2.
- [ ] Write `docs/eval_results.md` with tables, failure analysis, limitations.
- **Done when:** one command reproduces the numbers in the README (thanks to the cache).

### M4 Tests and CI finish (3 to 4 h)
- [ ] Fill any gaps against the section 10 table. Ruff clean. CI green. Badge added.
- [ ] Confirm CI runs with **no** secrets set.

### M5 API and review page (5 to 7 h)
- [ ] `api/app.py` with the routes in section 11. `test_api.py`.
- [ ] `review.html`. Try the full human loop: upload a bad document, see it in the queue, correct it, see the new verdict.
- [ ] CSV export.

### M6 README, docs, polish (3 to 4 h)
- [ ] README (outline in section 15). Mermaid architecture diagram in `docs/architecture.md`.
- [ ] A 30 to 60 second screen recording or two screenshots (optional but helpful).
- [ ] Clean commit history. Check no key ever entered history (see section 14).
- [ ] Tick the Definition of Done (section 17).

### Cut order if you fall behind
Never cut: ground truth, evaluation with holdout, tests, CI, README.
Cut in this order: (1) all stretch items, (2) CSV export, (3) review page styling (keep a bare table), (4) second model run, (5) extra API routes beyond ingest, queue, review. Even the minimum version is a credible project.

---

## 14. Security and hygiene checklist

- `.env` is in `.gitignore` **before** the first commit. `.env.example` has no real key.
- If a key is ever committed or pasted publicly: revoke it in AI Studio and create a new one. Deleting the commit is not enough.
- Synthetic data only. Footer on every file.
- Validate file extension and size on upload. Save with a generated name, never trust the uploaded file name for paths.
- Do not log document contents or the key.
- Pin dependency versions in the requirements files.
- Before making the repo public on your resume, run a quick search of the repo history for the first characters of your key.

---

## 15. README outline

1. One-sentence summary and CI badge.
2. The problem (and that it's a personal project with synthetic data, **not** the INAEC system).
3. How it works: architecture diagram (Mermaid) and the stage table.
4. Design decisions and trade-offs: AI reads, code decides; human review; raw output stored; cache; reference date parameter; append-only audit log.
5. Results: evaluation tables (dev and holdout), false clears list, link to `docs/eval_results.md`.
6. Limitations (be specific: synthetic, 30 documents, one model family, thresholds are illustrative).
7. Quick start in a handful of commands: clone, venv, install, add key, `init-db`, `process`, `serve`, `pytest`.
8. Project structure.
9. What I'd do next.

---

## 16. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Model has no free quota on your account ("limit: 0") | Probe on Day 0. Try another Flash-Lite model. Enable billing for pennies. Or build the regex baseline first |
| Free daily quota runs out mid-evaluation | Cache. Resume next day (midnight Pacific is about 3 to 4 PM Manila) |
| Per-minute 429 errors | 7 s spacing and backoff |
| Google changes model names or deprecates one | Model name in config; adapter isolates the SDK; Day-0 check |
| Two API styles confuse you | Probe decides; only `gemini.py` knows |
| Structured output returns nothing when truncated | Generous `max_output_tokens`; always validate yourself; record failures |
| Scans too blurry, accuracy poor | Good: that's a finding. Report it. Lower the degradation if every scan fails |
| Scope creep | Cut order in section 13 |
| AI writes code you can't explain | Understanding checks; prefer plain code; explain-back rule |
| Circular grading | Ground truth written by hand, separate from rules code |
| Overfitting the prompt to 30 documents | Dev/holdout split; report limitation |
| Leaked key | Section 14 |
| Quitting half-done | Vertical slices: after M2b you already have a working demo |

---

## 17. Definition of Done (gate for the resume)

- [ ] Public GitHub repo with README, MIT license, clear structure.
- [ ] 30 synthetic documents + 2 duplicates + `ground_truth.csv` (hand-written, with dev/holdout split) committed, and a seeded generator script.
- [ ] `process` handles a batch end to end; duplicates are skipped; re-running changes nothing.
- [ ] Rules engine is a pure function with tests for every status and boundary.
- [ ] At least 25 passing tests; no network needed; CI green with badge.
- [ ] Evaluation reproduces your numbers with one command; both dev and holdout reported; failure analysis and limitations written.
- [ ] Review page works for the full human loop; audit log is append-only (tested).
- [ ] No secrets in repository history.
- [ ] You can answer every question in section 18 aloud without notes.

---

## 18. Interview preparation (plain-language answers)

1. **Why not let the AI decide if it's expired?** AI is probabilistic. It can misread or invent a date. Date maths is exact in plain code, easy to test and easy to explain. So the AI only reads, and code decides.
2. **What happens when the AI is unsure or wrong?** Code checks the output (valid dates, expiry after issue, required fields). Anything doubtful goes to a human review queue instead of being auto-approved.
3. **How did you measure accuracy?** I wrote the correct answers for 30 synthetic documents myself, graded field by field, tuned prompts on 20 and checked on 10 held-out ones. I report counts, the failures, and the limits of such a small test.
4. **What was your worst failure?** Use your real one from `eval_results.md`.
5. **Why did you mock the AI in tests?** So tests are fast, free, repeatable and never need an API key in CI.
6. **What is idempotent about it?** Every file gets a fingerprint. Running the same folder twice doesn't create duplicates.
7. **Why SQLite and plain SQL?** It's simple, needs no server, and I can explain every table and join. A production system would likely use Postgres.
8. **What is the audit log for?** To show who did what and when. It's append-only; database triggers block edits and deletes.
9. **What would you change for production?** Real documents with consent and security review, a managed database, authentication, monitoring, a larger and messier test set, retries and queues.
10. **How is this related to your INAEC work?** It's the same kind of problem (tracking expiring records), but a personal project built on fake documents. At INAEC I automated pilot records with Google Apps Script.

---

## 19. Resume integration (only after section 17 is ticked)

Replace the bracketed items with **your real numbers**. Do not publish a bullet until the number exists.

- Built **OpsAudit**, a document-expiry triage pipeline in Python (FastAPI, SQLite) that extracts fields from synthetic compliance documents with a Gemini model and applies deterministic compliance rules; correct on **[X]/[Y]** held-out documents.
- Compared two prompt versions against a hand-labeled ground-truth set, raising field accuracy from **[A]/[B]** to **[C]/[D]**; reported all false clears.
- Wrote **[N]** Pytest tests with a mocked AI client and GitHub Actions CI on every push.
- Added a human-review queue with an append-only audit log.

Keywords you may add **once true**: Python, FastAPI, SQL/SQLite, Pydantic, Pytest, GitHub Actions (CI), REST APIs, LLM structured output, prompt evaluation, data pipelines, automation. Only add "bug tracking" if you really kept GitHub Issues.

AI/data/automation resume version changes (when you're ready): headline "Software Developer | Automation and Data Workflows"; summary leads with Apps Script automation, SQL and this project; skills list Python and SQL first; this project goes first under Projects, Botika shortened; INAEC pilot records bullet gets the most space.

Add a link to the repo in the header or the project line. Label it clearly as a personal project with synthetic data.

---

## 20. Day-0 checklist (what the plan could not verify for you)

Settle these in Milestone 0 before writing any real code:

1. Which Gemini model works on your free key, and its real daily and per-minute limits.
2. Whether Interactions or `generate_content` works with your SDK version, and the exact call for attaching a PDF or image.
3. Whether nullable fields in the schema work or need the empty-string fallback.
4. The current recommended major versions of the GitHub Actions in `ci.yml`.
5. Which operating system and shell you use, so every command can be given in the right form.
6. Whether Pillow's `load_default(size=...)` works on your version (otherwise bundle a font file).

If any answer contradicts this plan, change the plan, not the facts.
