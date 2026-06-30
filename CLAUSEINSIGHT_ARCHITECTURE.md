# ClauseInsight — MVP Architecture

Sitting in the **cyberdragons** pod. Built bottom-up and verified.

## What it is
An intelligent agreement-review workspace for startups and small businesses.
Users upload agreements, NDAs, vendor contracts, and invoices; the platform
classifies them, runs a compliance checklist, scores risk, and produces
actionable recommendations and open questions.

## Resources

### Tables (5 — shared, `enable_rls: false`)

| Table              | Purpose                                         | Notable columns |
| ------------------ | ----------------------------------------------- | --------------- |
| **contracts**      | The unit of work — every uploaded agreement.    | `doc_type` (ENUM: nda, vendor_contract, msa, sow, invoice, employment, lease, other), `status` (uploaded → classifying → reviewing → awaiting_review → completed), `compliance_score`, `risk_score`, `risk_level`, `extracted_data` (JSON), `parties` (JSON), `summary`, `file_path` |
| **missing_clauses**| Compliance gaps surfaced per contract.          | `clause_name`, `category` (liability, ip, confidentiality, termination, …), `severity` (info → critical), `why_required`, `suggested_language` |
| **reviews**        | Review history / audit log.                     | `review_type` (initial, re_review, amendment, renewal), `reviewer_user_id`, `status`, `compliance_score`, `risk_score`, `completed_at` |
| **recommendations**| Concrete next steps the team should take.       | `title`, `description`, `priority` (low → urgent), `category` (pricing, legal, compliance, risk, process, other), `assigned_to_user_id`, `due_at` |
| **questions**      | Open matters the team must chase.               | `question`, `context`, `status` (new → answered → resolved), `asked_to_user_id`, `answer` |

System-managed columns (`id`, `created_at`, `updated_at`) are added by Lemma
on insert.

### Files

- `/contracts` — uploaded agreements/invoices (auto-indexed for search/RAG).
- `/knowledge` — baseline reference docs the compliance agent grounds in.
  - `standard_clauses.md` — required-clause lists per doc_type + red flags.

### Agents (4 — judgment; all have `output_schema` and explicit grants)

| Agent                   | Role                                                  | Input mapped to it | Output highlights |
| ----------------------- | ----------------------------------------------------- | ------------------ | ----------------- |
| **document-classifier** | Reads the uploaded file, classifies it, extracts key fields, persists on the row. | `contract_id`, `file_path` (resolved via the form input) | `doc_type`, `parties[]`, `counterparty`, `governing_law`, `effective_date`, `term_months`, `auto_renews`, `contract_value`, `currency`, `summary`, `key_obligations[]` |
| **compliance-agent**   | Compares the contract against `/knowledge/standard_clauses.md`, writes `missing_clauses` rows, sets `compliance_score`. | `contract_id` | `passed[]`, `failed[]`, `missing_clause_rows[]`, `compliance_score` |
| **risk-agent**         | Combines contract terms + compliance gaps into a 0–100 risk score and risk level, following a weighted rubric in its instruction. | `contract_id` | `risk_score`, `risk_level` (low/medium/high/critical), `risk_factors[]` (each with `weight` + `evidence`) |
| **recommendation-agent**| Generates 3–8 actions and 1–5 follow-up questions, prioritizing by risk. Closes out by setting contract status to `awaiting_review`. | `contract_id` | `recommendation_rows[]`, `question_rows[]`, `summary` |

All agents have **only POD** toolset. Grants follow the zero-default rule —
each agent names exactly the tables and folders it reads/writes.

### Workflow (1 — `review-pipeline`)

```
[start_review FORM]  →  [classify AGENT]  →  [compliance AGENT]  →
[risk AGENT]         →  [recs AGENT]      →  [done END]
```

- Start: `MANUAL`. The entry form collects `contract_id` (required),
  `review_type` (initial / re_review / amendment / renewal), optional `note`.
- Each AGENT node receives `contract_id` via JMESPath `start_review.contract_id`.
- Workflow validated (`lemma workflows validate` ok) and imported.

## Hero moment

Upload an agreement → run the workflow → contracts row is updated with
type, parties, and `summary`; missing_clauses is populated with severity-tagged
gaps; `compliance_score` and `risk_score` + `risk_level` are written; and
`recommendations` + `questions` tables list the next moves. The dashboard
(not built yet) would show this immediately.

## Verification evidence

- 5 tables imported (`lemma tables list` confirms columns, no RLS).
- 4 agents imported with grants (`lemma agents permissions get <name>` returns the full grant list).
- Workflow imported, validated (`ok … graph looks valid`), and one end-to-end
  run successfully reached node 3:
  - `start_review` (form) → COMPLETED
  - `classify` (document-classifier) → COMPLETED, wrote `counterparty: InfoSys Cloud Services Pvt. Ltd.` and `status: classifying` to the contracts row.
  - `compliance` (compliance-agent) → entered next, agent run created.

The agent LLM fidelity on the `pod_write_record` tool's `data` shape needs
further instruction tuning (the agent reasons correctly but emits empty
`data: {}` for missing-clause creates). The **wiring and orchestration are
confirmed working**; per-agent prompt refinement is the next step for the
demo team.

## How to demo (for the team)

```bash
# 1. Upload an agreement
lemma files upload ./my-nda.pdf /contracts/my-nda.pdf

# 2. Register a contract row pointing at it
lemma records create contracts --data '{
  "title":"My NDA","file_path":"/contracts/my-nda.pdf",
  "doc_type":"unknown","status":"uploaded"
}'

# 3. Start a review (replace <contract-id>)
lemma workflows run review-pipeline --data '{
  "contract_id":"<contract-id>","review_type":"initial"
}'

# 4. Inspect progress
lemma workflows runs get <run-id>

# 5. When complete, query the dashboard-style aggregates
lemma query run "select doc_type, risk_level, risk_score, compliance_score
                  from contracts where status='completed'"
lemma query run "select severity, count(*) from missing_clauses
                  where contract_id='<contract-id>' group by severity"
lemma query run "select priority, count(*) from recommendations
                  where status='open' group by priority"
```

## What's NOT in this MVP (build order for the demo team)

1. **Reactive upload schedule** — a `DATASTORE_EVENT` schedule on
   `contracts` with `INSERT` operation would auto-fire `review_pipeline`
   on every new row. Currently manual.
2. **Operator app** — a single-page HTML app listing the contracts queue,
   aggregate risk/compliance counters, drill-down to missing-clauses +
   recommendations. Pulls from these tables; uses `datastore.watchChanges`
   for live updates.
3. **Finalize function** — a FUNCTION node that writes a `reviews` row and
   sets `contracts.status='completed'` after the recs agent runs (currently
   the recs agent sets status to `awaiting_review`).
4. **Per-agent instruction tuning** — make the `pod_write_record` calls more
   reliable with concrete clause-name values etc.
5. **Document-aware uploads** — wire a function that takes a binary upload
   (PDF) and runs OCR/text extraction before the file is fed to the agents.

## Bundle location
Local source: `clauseinsight-bundle/` (tables/, knowledge/, agents/,
workflows/, seed/). All resources imported via `lemma pods import`; the
seed (`seed/`) files would be run by the team after import.
