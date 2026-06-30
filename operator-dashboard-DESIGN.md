# ClauseInsight Operator Dashboard — DESIGN (v2)

## Single artifact
- `operator-dashboard.html` — single-page HTML app, no build, served at
  https://operator-dashboard-019f13ff.apps.lemma.work/
- `operator-dashboard.json` — bundle manifest (`type=html`, `html_file=...`,
  `visibility=POD`)

## Persona & purpose
A legal/contracts **operator** opens this and (a) drops a new contract PDF
into the intake, then (b) watches the review pipeline classify it, score it,
and feed the rest of the dashboard tabs with results — without leaving the
screen.

## Tabs (left → right)
1. **Intake** — drop zone + recent uploads + "what just happened" pipeline map.
2. **Contracts** — queue of every contract with status, risk, compliance,
   value, last update; filter by status × risk.
3. **Risk Distribution** — average risk & compliance KPIs + bucketed bars.
4. **Missing Clauses** — every gap the compliance agent flagged, with
   severity, category, "why required", and suggested language.
5. **Recommendations** — ranked by priority (urgent→low), with due dates.
6. **Review History** — every completed review with score deltas + decision
   reasoning.

## Single-page flow: PDF upload → dashboard

```
   USER drops PDF
        │
        ▼
   1. Storage:  client.files.upload(file, { directoryPath: "/me/contracts" })
        │   → /me/contracts/<safe-name>.pdf   (auto-indexed, searchable)
        ▼
   2. Register: client.records.create("contracts", {
                     title, file_path, doc_type:"unknown", status:"uploaded",
                     summary:"Awaiting pipeline…"
                  })
        │   → new contracts row in the table the dashboard is watching
        ▼
   3. Kick:      client.workflows.runs.create("review-pipeline")  → WAIT
        client.workflows.runs.submitForm(runId, {
          node_id: "start_review",
          inputs:  { contract_id, review_type:"initial", note? }
        })
        ↓
        ┌──────────────────────────────────────┐
        │ review-pipeline workflow             │
        │   ↓ start_review    (FORM → inputs)  │
        │   ↓ classify        (agent)          │  ← document-classifier
        │     → doc_type, parties, dates,     │
        │       contract_value, jurisdiction  │
        │   ↓ compliance     (agent)           │  ← compliance-agent
        │     → missing_clauses rows          │
        │     → contract.compliance_score     │
        │   ↓ risk            (agent)          │  ← risk-agent
        │     → contract.risk_score           │
        │     → contract.risk_level           │
        │   ↓ recs            (agent)          │  ← recommendation-agent
        │     → recommendations rows          │
        │     → reviews row                   │
        └──────────────────────────────────────┘
        │
        ▼
   4. UI lights up automatically:
        client.datastore.watchChanges() is registered on all 4 tables
        (contracts, missing_clauses, recommendations, reviews).
        Every agent-driven write fires a WSS event → dashboard re-renders.
        No refresh, no polling. The Intake tab shows a "running…" pulsing
        pill for in-flight contracts; the Contracts/Risk/Clauses/Recs/History
        tabs fill as rows land.
```

## Intake tab (default landing)
- **Dropzone** — drag-and-drop or click. PDF only. File goes to
  `/me/contracts/<sanitized>.pdf`.
- **Inline progress** — label "Uploading… 25% / Registering… 60% /
  Starting pipeline… 85%" with a thin accent bar. Toast in the topbar on
  success (`Uploaded · review pipeline started.`) or failure
  (`Upload failed: ...`).
- **Recent uploads list** — one row per local upload: PDF chip, name,
  current pipeline stage (running → complete/failed), link into the
  Contracts tab. A "Re-run review" button re-kicks the workflow against the
  existing `contract_id` (uses the same FORM submit-form pattern).
- **Pipeline map** — a static 7-step explainer telling the operator what
  happens after the upload (helpful when someone unfamiliar opens the app).

## SDK calls (the exact three used by the dashboard)
- **Upload** — `client.files.upload(file, { directoryPath: "/me/contracts",
  searchEnabled: true })`. Returns the indexed `{ id, path, ... }`.
- **Register** — `client.records.create("contracts", { title, file_path,
  doc_type: "unknown", status: "uploaded", summary: "…" })`. Returns the new
  `{ id, ... }`.
- **Kick** — `client.workflows.runs.create("review-pipeline")` returns
  `{ id, active_wait: { node_id: "start_review", wait_type: "HUMAN" } }`;
  `client.workflows.runs.submitForm(runId, { node_id: "start_review",
  inputs: { contract_id, review_type: "initial", note? } })` fires the pipeline.

(The existing dashboard also runs `client.datastore.watchChanges(...)` on all
4 tables so any downstream writes — agent-derived contract field updates,
new `missing_clauses` / `recommendations` / `reviews` rows — appear live.)

## State map
|Maps |Holds|
|---|---|
|`state.contracts` | id → contract row (load + watch) |
|`state.missing_clauses` | id → clause row |
|`state.recommendations` | id → rec row |
|`state.reviews` | id → review row |
|`state.contractTitleById` | contract id → title (joins keys) |
|`state.intakeRuns` | contract id → `{ run_id, status, started_at, file_path }` |
|`state.intakeFileMeta` | file_path → `{ name, contract_id, run_id, stage }` |
|`state.intakeBusy` | single-flight lock |
|`state.liveSubs` | array of WSS handles to close on tab unload |

## Scenarios
1. **Operator uploads a fresh MSA.** Drops PDF → 25% “Uploading…” → 60%
   “Registering contract row…” → 85% “Starting review pipeline…” → 100% +
   "Done". A pulse pill appears in "Recent uploads". Contracts tab shows
   status=uploaded→classifying→awaiting_review as agents finish.
2. **Pipeline succeeds.** Risk Distribution's Average Compliance / Risk re-
   averages with the new row. Missing Clauses tab gets the agent's clauses.
   Recommendations shows the new urgent/medium recs. Review History adds
   one new row.
3. **Operator hits Re-run on an in-flight upload.** The dashboard re-submits
   the same `contract_id` to the FORM node of `review-pipeline`, kicking a
   new run. No duplicate file upload.
4. **Operator opens the app fresh.** `/me/contracts/` is the source of
   truth for files; the dashboard's existing records watch rehydrates
   `state.*` on boot to show the latest snapshot.
5. **Operator drags a non-PDF.** Toast: `Only PDF files are accepted.`
   No row created. No workflow kicked.
6. **Operator drops while another upload is mid-flight.** Toast:
   `Another upload is in progress — queued.` Single-flight lock prevents
   overlapping writes.

## States per tab
- **Empty (first deploy):** "Nothing uploaded yet — drop a PDF above to
  begin." plus a "How the pipeline runs" sidebar.
- **Loading:** Instrinsic since all 4 tables are listed on a single
  promise — first paint shows no flash.
- **Has data:** Recent uploads list + Contracts/Risk/Clauses/Recs/History
  populated from the live watch.

## Permissions / grants
The operator's session owns `/me/contracts` (`folder.write`) and reads `POD`
files. The dashboard never calls the SDK as agent — it calls as the user
who owns `/me`. Workflow execution (`workflows.runs.create`,
`workflows.runs.submitForm`) needs only `workflow.execute` pod-wide; the
existing `review-pipeline` is `mode=GLOBAL` and open to POD members.

## Verification (this build)
- `lemma apps get operator-dashboard` ⇒ `status READY`,
  `release 019f16d2-2bea-71db-aa83-257605068241`.
- Drop a button into the dropzone, `lemma files upload
  /workspace/.../BlueOrbit_MSA_v2.pdf /me/contracts/BlueOrbit_MSA_v2.pdf` ⇒
  `status=PROCESSING → COMPLETED`.
- A `contracts` row appears with `file_path=/me/contracts/BlueOrbit_MSA_v2.pdf`
  and `status=uploaded`.
- `client.workflows.runs.create("review-pipeline")` → WAITING on
  `start_review`. `submit-form` ⇒ `status=RUNNING current=classify`.
- After ~1m: classify completes (`doc_type=msa`,
  `counterparty="BlueOrbit Cloud Services, Inc."`, `term_months=24`,
  `contract_value=480000`, `compliance_score=50`, `extraction_confidence=0.95`),
  then compliance (adds compliance_score, planned clause rows), then risk
  (adds risk_score/risk_level), then recs (adds rec rows + a reviews row).
- The dashboard's `client.datastore.watchChanges` on `contracts`,
  `missing_clauses`, `recommendations`, `reviews` light up each row as the
  agent writes land.
