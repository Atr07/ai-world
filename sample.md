The data side of Evaluation is complete: catalog, playbook and datasets. Next comes **Readiness Runs**, which ties them together. It runs an agent's resolved playbook against a pinned dataset version and produces a scored verdict.

## How a run works

1. **Pick** an agent and one of its datasets. The run pins the **dataset version** and takes a **snapshot of the resolved playbook**: every check with its version, weight, threshold and gate, plus the category weights. Later edits never change a past run.
2. **Invoke** the agent's AEI `/invoke` endpoint for each case, with limited concurrency, a timeout and retries.
3. **Score** each case against the checks that apply to it.
   - **Rule and code** checks run in this module.
   - **Prompt and hybrid** checks are marked "pending judge" until the LLM judge phase.
4. **Aggregate** the results:
   - Each category's score is the average across the cases that tested it.
   - The run score combines category scores using the category weights.
   - Failing any **must-pass** check means **not ready**, whatever the score.
   - Agent outages (timeouts, 5xx errors) are left out of the score. Too many of them makes the verdict **incomplete**, not a fail.

There are four phases, and the standing rules still apply.

---

## Phase R1: Scoring engine (pure logic, no API)

```text
PHASE R1 - READINESS RUNS: SCORING ENGINE (pure functions, no DB, no HTTP)
Implement execution of rule and code checks and the aggregation, as pure, fully unit-tested code
in app/services/scoring/.

1. Inputs: a ScoringCase (case_ref, input_text, expected_output, metadata, target_check_keys,
   response text, response_json = parsed response if it is valid JSON else None) and a
   ResolvedCheckSnapshot (check_id, key, version, type, category_key, definition, weight,
   threshold, hard_gate).
2. Applicability: a check applies to a case unless the case's target_check_keys is non-empty and
   doesn't contain the check key.
3. Rule engine: evaluate each condition on field "response", "expected" or "json:<dot.path>" (a
   path into response_json; missing path = condition fails with evidence "path_not_found").
   applies_when skips conditions whose metadata doesn't match; all skipped -> result
   "not_applicable". combine all/any. Score 1.0 or 0.0. Evidence per condition: operator,
   expected value(s), what was found (truncate to 200 chars), passed/skipped.
4. Code evaluators: implement every registered evaluator (exact_match vs expected, regex,
   keyword_presence, forbidden_pattern, numeric_range, json_field_match, field_extraction vs the
   expected output parsed as JSON - if expected isn't valid JSON, result "error" with a clear
   message). Return score 0-1 plus evidence; apply invert_score. Regex runs with a timeout or a
   pattern-length/complexity guard to avoid catastrophic backtracking.
5. Prompt checks, and the deep part of hybrid checks: result status "pending_judge" (no score) in
   this phase. Hybrid: run the surface part now; if surface_threshold is set and the surface score
   is below it -> final status "scored", score = surface score, passed false, evidence
   "failed_surface". Otherwise "pending_judge" with the surface score kept in evidence.
6. CheckResult: status in {scored, not_applicable, pending_judge, error}, score, passed
   (score >= threshold), evidence, duration_ms. Errors never raise out of the engine.
7. Case aggregation: case category score = weight-weighted mean of scored checks in that category;
   case passed = every scored check passed; hard_gate_failures = keys of hard-gate checks with
   status scored and passed false.
8. Run aggregation (only over cases with status "invoked"; infra-error cases are excluded and
   counted):
   - category_scores[c] = mean of case category scores over cases where c had scored checks.
   - composite = sum(category_scores[c] * weight[c]) / sum(weight[c]) over categories present
     (category weights from the snapshot).
   - verdict: "incomplete" if any pending_judge results exist, or if
     infra_errors / total_cases > max_infra_error_rate (default 0.1), or if nothing was scored;
     else "not_ready" if any hard-gate failure or composite < pass_threshold; else "ready".
   - Also return per-check stats {key, scored, passed, pass_rate, avg_score} and counts.
9. Tests: every operator and evaluator (pass, fail, edge cases), applies_when, json paths,
   invert_score, regex guard, hybrid surface threshold, applicability, case and run aggregation
   with hand-calculated expected numbers, all verdict paths, infra exclusion.
Run full pytest, list changed files, then STOP. No migration in this phase.
Commit as "feat(eval): readiness scoring engine for rule and code checks".
```

## Phase R2: Run model, orchestration and API

```text
PHASE R2 - READINESS RUNS: MODEL, ORCHESTRATION AND API
1. Models (Uuid ids/FKs, no cascades, all constraints named; one migration "add readiness runs"
   that renders offline for mssql):
   - EvalRun (eval_runs): agent_id FK agents, hierarchy_node_id FK (agent placement at start),
     dataset_id FK, dataset_version_id FK, status in {queued, running, completed, failed,
     cancelled}, verdict in {ready, not_ready, incomplete} null, pass_threshold Float,
     composite_score Float null, category_scores JSON, check_stats JSON, counts JSON
     {total, invoked, infra_errors, agent_errors, passed, failed}, playbook_snapshot JSON (resolved
     checks with version, definition, weight, threshold, hard_gate, category) and
     category_weights JSON, progress fields total_cases / completed_cases, error_message Text null,
     notes Text null, started_at, finished_at, created_by, timestamps.
   - EvalRunCase (eval_run_cases): run_id FK, case_id FK golden_cases, case_ref, status in
     {pending, invoked, infra_error, agent_error}, response_text Text null, latency_ms, attempts,
     error Text null, category_scores JSON, passed Boolean null, hard_gate_failures JSON.
     UniqueConstraint(run_id, case_id).
   - EvalRunCheckResult (eval_run_check_results): run_case_id FK, check_id FK, check_key,
     check_version, category_key, status, score Float null, passed Boolean null, evidence JSON,
     duration_ms. Index on (run_case_id).
2. Starting a run, POST /api/v1/eval/runs (EVALS_RUN), body {agent_id, dataset_id,
   dataset_version_id?, pass_threshold?, notes?}. Validate: agent exists, is active and placed
   ("agent_not_placed"); the dataset is available to the agent (placed on its node or an
   ancestor, or owned by it) and active ("dataset_not_available"); version ready (default =
   current); the playbook resolves at least one active check ("playbook_empty"); no queued or
   running run for the same agent and dataset ("run_already_active"). pass_threshold default from
   settings READINESS_PASS_THRESHOLD (0.8). Snapshot the playbook and category weights, create the
   run and its pending cases, submit to the existing TaskRunner, return 202.
3. Execution (app/services/readiness_runs.py):
   - Invoke the agent's /invoke with the existing AEI client and request schema (check
     agents/aei for the exact fields; send the case's input_text and, if the schema supports it,
     reference_context and metadata). The response field is a string; parse JSON for
     response_json.
   - Concurrency READINESS_CONCURRENCY (default 4), per-case timeout READINESS_TIMEOUT_SECONDS
     (default 60), retries READINESS_MAX_RETRIES (default 2) with backoff only for timeouts,
     connection errors and 5xx -> infra_error after retries; 4xx or a contract violation ->
     agent_error (counted as a failed case, not excluded).
   - Score each case with the R1 engine, store results, update completed_cases. Commit in small
     batches; a cancelled run stops picking new cases.
   - Finish: run aggregation -> composite, category_scores, check_stats, counts, verdict, status
     completed. Unexpected exception -> status failed with error_message (no stack trace in the
     API).
   - On app startup, mark queued/running runs as failed with error_message "interrupted by
     restart" (the in-process runner doesn't survive App Service restarts).
   - Don't log case inputs, responses or expected outputs above DEBUG.
4. Endpoints (prefix /api/v1/eval/runs; reads for any principal, writes EVALS_RUN, audited):
   GET / (filters agent_id, dataset_id, status, verdict; paginated, newest first), GET /{id}
   (summary + snapshot + agent/dataset refs), GET /{id}/cases (paginated; filters status, passed,
   q), GET /{id}/cases/{run_case_id} (with check results), POST /{id}/cancel,
   GET /api/v1/agents/{id}/runs.
5. Setup step "readiness_run" (target "readiness_runs"): complete when at least one run completed.
6. Tests with an httpx MockTransport fake agent (no real network): happy path, verdicts ready /
   not_ready / incomplete (prompt check pending), infra errors excluded and retried,
   agent_error counted, timeout, cancel, the active-run conflict, every validation error,
   snapshot unaffected by later check/playbook edits, the startup recovery, permissions.
Run full pytest, tests/test_migrations.py, offline mssql render (both directions), alembic check,
show alembic heads, list changed files, then STOP.
Commit as "feat(eval): readiness runs orchestration and API".
```

## Phase R3: frontend-local pages and docs

```text
PHASE R3 - READINESS RUNS UI (frontend-local) AND DOCS
Interim UI rules as before. Do NOT modify frontend/.
1. Enable "Readiness Runs" in the sidebar (/certify/runs); setup target "readiness_runs" maps here.
2. List: filters agent, dataset, status, verdict; table with agent, dataset vN, started, duration,
   status (with progress %), composite %, verdict pill (Ready / Not ready / Incomplete).
3. "New run" dialog: agent select (active, placed) -> dataset select (from
   GET /agents/{id}/golden-datasets) -> version select (default current) -> pass threshold % ->
   notes. Show the resolved check count for the agent before starting.
4. Run detail /certify/runs/[id]: poll every 3 s while queued/running; header with verdict,
   composite %, threshold %, counts; category score bars; hard-gate failures listed first;
   per-check table (key, type, category, pass rate, avg score, gate, pending-judge count); cases
   table (case_ref, status, passed, latency) with a drawer showing input, expected, response and
   each check's result and evidence; Cancel button while running; an "Incomplete" banner that
   explains why (pending judge checks / too many agent errors).
5. Agent Registry card: latest run verdict and a link to its runs.
6. docs/api/readiness-runs.md: lifecycle, snapshot/reproducibility, scoring formulas with a worked
   example, verdict rules, settings, error codes, endpoints.
7. frontend-local lint, typecheck, build; list changed files; STOP.
Commit as "feat(eval): readiness runs interim UI and docs".
```

## Phase R4: Verification

```text
PHASE R4 - VERIFICATION
1. Full pytest, migrations round trip, alembic check, one head, offline mssql render review of
   the new tables (named constraints, index sizes, NVARCHAR(max) for JSON/Text).
2. Local end-to-end: start the credit_prequalification (or claims_triage) reference agent locally,
   register/place it at Lending, run against banking.lending.core, and report the verdict, the
   composite and one case's check evidence.
3. Confirm no case content is logged at INFO, and settings READINESS_* are documented in
   .env.example.
4. Final report with deviations and follow-ups (LLM judge for prompt/hybrid checks; Azure: the
   App Service must be able to reach agent endpoints; the in-process runner limits; a durable
   queue later if runs get long).
```

---

**What to expect when you test:** with only rule and code checks assigned, a run gives a real **Ready / Not ready**. If any prompt or hybrid checks are assigned (like Answer correctness or Fair lending), the verdict will be **Incomplete**, which is correct. The **LLM judge** phase turns those into real scores using Azure OpenAI; I'll write it after R4.

To see a decisive verdict now, temporarily **disable** the prompt checks at Lending in the playbook and run again. That also exercises the disable feature.

