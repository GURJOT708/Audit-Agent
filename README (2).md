# Ghost Data Audit Agent

An agent that audits whether a user's personal data was **really** deleted at Northstar Shop, built for the *Ghost Data Still Exists* challenge (HackerRank Orchestrate, October 2026).

For each deletion request the agent decides one of three outcomes and produces an ordered step plan:

| Status | Meaning | `steps_json` |
| --- | --- | --- |
| `COMPLETE` | Deletion is truly finished | Verification steps only, each citing evidence |
| `INCOMPLETE` | Data, a data flow, or a restore path remains | Findings, then remediation, then verification |
| `NOT_APPLICABLE` | The person never asked for deletion | `[]` |

> **Status of this repo:** `Audit_agent.ipynb` currently contains the data exploration and a **heuristic baseline** that produces a valid-shaped `output.csv`. It does **not** yet parse the evidence files or apply the deletion rules. See [Current state and limitations](#current-state-and-limitations) and [Target approach](#target-approach) for what is done and what is next.

---

## Repository layout

```text
.
├── Audit_agent.ipynb          # exploration + baseline pipeline (this notebook)
├── README.md
├── AGENTS.md / CLAUDE.md      # challenge instructions for AI coding tools (session logging etc.)
├── dataset/
│   ├── deletion_requests.csv         # 518 cases to audit
│   ├── sample_deletion_requests.csv  # 32 sample cases
│   ├── sample_output.csv             # correct answers for the samples
│   ├── user_identities.csv           # 850 current identities
│   ├── user_audit_log.csv            # 306 historical identity changes
│   ├── system_activity.csv           # 4,011 operational events
│   ├── deletion_rules.csv            # 15 rules: required treatment per data category
│   ├── evidence_files.csv            # manifest of 94 evidence files
│   ├── output.csv                    # blank submission template
│   └── evidence/                     # PDFs, images, logs, JSON, SQL snapshots
├── output.csv                 # generated predictions (repo root)
└── evaluation/
    ├── evaluation_report.md
    └── usage_report.md
```

The notebook reads files from a flat `/content/` folder (Google Colab). If you run it elsewhere, change the paths (see below).

---

## Setup

### Option A: Google Colab (how the notebook was written)

1. Open `Audit_agent.ipynb` in Colab.
2. Upload these files into `/content/`:
   `deletion_requests.csv`, `sample_deletion_requests.csv`, `sample_output.csv`, `user_identities.csv`, `user_audit_log.csv`, `system_activity.csv`, `deletion_rules.csv`, `evidence_files.csv`, `problem_statement.md`.
3. Upload the whole `evidence/` folder to `/content/evidence/` (zip it, upload, and unzip with `!unzip evidence.zip -d /content/`). The manifest paths in `evidence_files.csv` are relative to this folder.
4. Run all cells (**Runtime → Run all**). The final cell writes `/content/output.csv`.

### Option B: Local machine

```bash
git clone https://github.com/interviewstreet/hackerrank-orchestrate-october26.git
cd hackerrank-orchestrate-october26

python3 -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install pandas jupyter

jupyter notebook Audit_agent.ipynb
```

Then replace the `/content/` prefix with your data folder, for example `dataset/`. The simplest way is to define one constant in the first cell and build every path from it:

```python
from pathlib import Path
DATA = Path("dataset")          # was "/content"
requests_df = pd.read_csv(DATA / "deletion_requests.csv")
```

Write the final predictions to `output.csv` in the repository root.

**Requirements:** Python 3.10+, `pandas`. The planned evidence parsers will additionally need a PDF text extractor, an OCR or vision model for images, and `sqlite3` (standard library).

> Never commit API keys. Load model credentials from environment variables and keep them out of `code.zip`.

---

## Notebook walkthrough

| Cells | What they do |
| --- | --- |
| 0–6 | Load the problem statement, rules, requests, sample output, identities and evidence manifest. Read the challenge `README.md`, `AGENTS.md` and `agent_trace_instructions.md`. |
| 7–22 | Search the Colab filesystem for the `evidence/` folder. **It was not found**; only `account_admin_console_20260926.png` had been uploaded. |
| 23 | Profile `system_activity.csv` (4,011 rows, 17 event types such as `DELETE_RECORD`, `SYNC_RECORD`, `RESTORE_BACKUP`, `DISABLE_DATA_FLOW`, `BLOCK_RESTORE`). |
| 24 | Check how many request emails match a current identity directly: **403 of 518**. The remaining cases need historical emails from the audit log. |
| 25 | Study `sample_output.csv` to learn the shape of the answers (systems like `account_service`, `security_service`, `analytics_platform`; resources like `account_005`, `analytics_901`). |
| 26 | Build an identity map from current identities plus `user_audit_log.csv` email changes: 957 unique emails mapped to `user_id`s. |
| 27 | Baseline generator that writes all 518 rows to `output.csv`, sorted by `case_id`. |

---

## Current state and limitations

The baseline in cell 27 works as follows:

1. **Applicability:** flags a request as `NOT_APPLICABLE` when the text contains words like *opt-out, access, export, correction* and none of *delete, remove, close, forget, erase*.
2. **Identity:** looks up `user_id` from the identity map built in cell 26.
3. **Status:** `COMPLETE` unless `reported_status` is `NOT_STARTED` or `PARTIALLY_COMPLETE`, then `INCOMPLETE`.
4. **Steps:** emits a single `account_service` record per case (`account_<n>`) with a fixed evidence ID (`evidence_54`).

This produces a well-formed file, but it is a **placeholder**, not an audit. Known gaps:

- **It trusts `reported_status`.** The challenge says that field is a claim, not proof. Real state must come from `system_activity.csv` and the evidence files.
- **No evidence parsing.** The `evidence/` folder was never available in the notebook run, so no log, JSON, SQL snapshot, PDF or image has been read.
- **`deletion_rules.csv` is not applied.** There is no DELETE / REDACT / ANONYMIZE / RETAIN decision logic, no tombstone handling, and no retained-record verification.
- **No data-flow or restore-risk detection**, so `DISABLE_DATA_FLOW`, `BLOCK_RESTORE` and their findings and verifications are never produced.
- **Guessed resource IDs.** `account_<n>` is derived from `user_<n>`, and if no identity matches, the code invents `user_<case number>`. This can produce wrong or fabricated IDs.
- **Single hardcoded evidence ID** for every case.
- **No evaluation yet.** Accuracy against `sample_output.csv` has not been measured, so no score is claimed here.

Treat the current `output.csv` as a format check only; it should not be submitted as a final answer.

---

## Target approach

The plan is mostly **deterministic code**, with an LLM used only where code cannot read the evidence (images, messy PDFs). This keeps results reproducible and token cost low.

```text
request ──► 1. classify request ──► NOT_APPLICABLE? ──► []
               │
               ▼
        2. resolve identity   (current + historical emails, user_id, email hash)
               │
               ▼
        3. collect evidence   (system_activity rows + parsed evidence files)
               │
               ▼
        4. reconstruct state at audit_at   (replay events in time order)
               │
               ▼
        5. apply deletion_rules.csv        (required vs actual treatment)
               │
               ▼
        6. build ordered steps_json        (findings → remediation → verification)
```

1. **Classify the request.** Use the request text (and `CLASSIFY_REQUEST` events in `system_activity.csv`) to decide whether deletion was actually requested.
2. **Resolve identity.** Map the email to a `user_id` using `user_identities.csv` and `user_audit_log.csv`, so old emails and hashes are covered.
3. **Collect evidence.** Filter `system_activity.csv` by the user's resources and identifiers. Parse each evidence type into one common record: *(system, resource, identifiers present, timestamp, evidence_id)*. SQL snapshots go through `sqlite3`, logs and JSON through plain parsers, PDFs through text extraction, images through OCR or a vision model.
4. **Reconstruct state at `audit_at`.** Replay events in time order. A delete followed by a `SYNC_RECORD`, `RESTORE_BACKUP` or re-index means the record is back. Treat `reported_result` values as claims to verify.
5. **Apply the rules.** Map each record's data category to its required action (`DELETE`, `REDACT`, `ANONYMIZE`, or retain) and compare with its actual state. The difference gives the findings and remediation. Tombstones may keep `user_id` but must lose name, email, phone and email hash. Retained records must never be deleted.
6. **Generate steps.** Emit exactly the allowed operation/state pairs, sorted by phase, then the operation order from the problem statement, then `system_id`, then `resource_id`, with consecutive `step_no`. Remediation and future verification steps in `INCOMPLETE` cases use an empty `evidence_id`.

---

## Output format

`output.csv` has exactly one row per request, sorted by `case_id`:

```csv
case_id,deletion_status,steps_json
case_000,NOT_APPLICABLE,[]
```

Each step in `steps_json`:

```json
{
  "step_no": 1,
  "phase": "FINDING",
  "system_id": "account_service",
  "resource_id": "account_005",
  "operation": "REPORT_REMAINING_RECORD",
  "state": "PRESENT",
  "evidence_id": "evidence_54"
}
```

Allowed phases: `FINDING`, `REMEDIATION`, `VERIFICATION`. Allowed operations and their states are listed in `problem_statement.md`.

Quick validity check before submitting:

```python
import json, pandas as pd
out = pd.read_csv("output.csv")
req = pd.read_csv("dataset/deletion_requests.csv")
assert len(out) == len(req) == 518
assert out["case_id"].is_monotonic_increasing
assert set(out["deletion_status"]) <= {"COMPLETE", "INCOMPLETE", "NOT_APPLICABLE"}
for _, r in out.iterrows():
    steps = json.loads(r["steps_json"])
    assert [s["step_no"] for s in steps] == list(range(1, len(steps) + 1))
    if r["deletion_status"] == "NOT_APPLICABLE":
        assert steps == []
```

---

## Evaluation

Planned workflow, to be filled into `evaluation/evaluation_report.md`:

1. Run the pipeline on `sample_deletion_requests.csv` and compare with `sample_output.csv`.
2. Report case-level status accuracy, step-level and field-level accuracy, and sequence accuracy.
3. Add at least three custom test cases (for example a user whose email changed, a tombstone case, and a restore-risk case).
4. Log every failure found and the fix applied.

`evaluation/usage_report.md` must summarize the final full run: model providers and names, call counts, input and output tokens, per-request averages, and estimated cost. If no LLM is used for a stage, state that explicitly.

---

## Submission checklist

- [ ] `output.csv` with 518 rows, sorted by `case_id`
- [ ] `code.zip` with runnable code, prompts, configuration, this README, and `evaluation/`
- [ ] `evaluation/evaluation_report.md`
- [ ] `evaluation/usage_report.md`
- [ ] Chat transcript and agent trace (see `agent_trace_instructions.md`)
- [ ] No API keys or credentials anywhere in the archive
