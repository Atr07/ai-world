Here are the remaining phases, rewritten with every decision so far. One thing to confirm first:

> **Hierarchy assumption: Option A.** A single root: **L0 Global** → **L1 Country** (Canada, US) → **L2 Line of business** (Banking) → **L3 Business function** (Lending). If your teammate confirms three separate L0 roots instead (Option B), tell me **before Phase 1b**, because that changes the single-root rule.

## How to run these

1. Paste the **Standing rules** block once at the start of each new Copilot chat.
2. Paste one phase at a time. Copilot stops after each one; check its report (tests, `alembic heads`, files changed), then paste the next.
3. Commit after each phase. Only `pytest` must pass between phases; the frontend comes in Phase 6.

---

## Standing rules (paste once per Copilot session)

```text
STANDING RULES FOR THIS WORK
- Monorepo: backend/ (FastAPI, SQLAlchemy 2.0 async, Alembic), frontend-local/ (Next.js interim UI
  used for testing), frontend/ (the team's React SPA: NEVER modify it).
- Follow existing conventions: routers app/api/v1, logic app/services, AppError subclasses ->
  {"detail","code"}, record_audit/diff_changes, LifecycleStatus/assert_transition,
  require_capability/PrincipalDep (capabilities HIERARCHY_ADMIN, EVALS_ADMIN, EVALS_AUTHOR,
  EVALS_RUN), Page/PaginationParams, JSONType, HierarchyService helpers. Thin routes, type hints,
  docstrings, no print().
- Nothing is seeded. All data is created through the API/portal.
- Units: weights and thresholds are fractions 0-1 everywhere (API and DB). UIs show x100.
- Databases: production Azure SQL (SQL Server), local SQLite acp_td.db. Portable types only
  (String(n) with indexed strings <= 450 chars, Text, Integer, Float, Boolean,
  DateTime(timezone=True), String(36) ids, JSONType). No ON DELETE CASCADE and no cascade
  relationships (enforce delete rules in services). No partial indexes or DB-specific defaults.
- Migrations: never edit migrations on origin/main (latest main revision: cbfebab50887). New
  migrations chain from the current head; exactly one head. Review each generated migration.
- After each phase: run the full pytest suite, show `alembic heads`, list changed files and
  decisions you made, then STOP and wait for me.
```

---

## Phase 1b: Phase 1 follow-ups (small, do first)

```text
PHASE 1b - HIERARCHY AND SETUP ADJUSTMENTS
1. Hierarchy code pattern: widen to ^[a-z][a-z0-9_-]{1,49}$ (hyphens allowed; path separator stays
   "."). Add tests for "retail-banking" valid and for the descendant query still excluding sibling
   prefixes (e.g. "bank" vs "bank-us").
2. Level labels constant (single source of truth): 0 "Global", 1 "Country", 2 "Line of business",
   3 "Business function". Add level_label to NodeRead, NodeTree and NodeRef; expose
   GET /api/v1/hierarchy/levels -> [{level, label}].
3. Single root stays: exactly one L0 node (service-enforced, tested).
4. Setup steps: replace the URL "link" with a "target" key. Targets: "hierarchy",
   "eval_catalog.categories", "eval_catalog.checks", "eval_playbook". Keep title, description,
   detail, complete. Update existing steps and tests, and map targets to routes in
   frontend-local's setup banner (hierarchy -> /admin/hierarchy, eval_catalog.categories ->
   /evaluation/catalog?tab=categories, eval_catalog.checks -> /evaluation/catalog?tab=checks,
   eval_playbook -> /evaluation/playbook).
Commit as "feat(hierarchy): allow hyphenated codes, add level labels and setup targets".
```

## Phase 2: Eval types and evaluator registry

```text
PHASE 2 - EVAL TYPES AND EVALUATOR REGISTRY (metadata the portal uses to render forms)
1. app/core/eval_types.py: StrEnum EvalType {prompt, code, rule, hybrid} (code constant, not a
   table) with label and description per type.
2. Definition schemas (Pydantic v2, extra="forbid"), one per type:
   - RuleDefinition: conditions: list[RuleCondition] (min 1), combine: "all" | "any".
     RuleCondition: field ("response" | "expected" | "json:<dot.path>"), operator in {contains,
     not_contains, contains_any, contains_all, equals, not_equals, regex, not_regex,
     number_between, length_between, json_field_equals}, value / values / min / max as the
     operator requires (validate per operator; regex must compile), case_sensitive: bool = false.
   - CodeDefinition: evaluator (a registered evaluator name), config (validated against that
     evaluator's config schema).
   - PromptDefinition: rubric (min 20 chars), criteria: list[str] = [], score_scale {min: 0,
     max: 1 by default}.
   - HybridDefinition: surface_eval (RuleDefinition or CodeDefinition), deep_eval
     (PromptDefinition), surface_weight 0-1 (default 0.3).
   Provide validate_definition(type, data) -> normalized dict raising ValidationError
   "invalid_definition" with field-level details.
3. app/services/evaluators/registry.py: a registry of deterministic evaluators with name, label,
   description and a config schema (Pydantic model): exact_match {case_sensitive, strip},
   regex {pattern, must_match}, keyword_presence {keywords, mode any|all, case_sensitive},
   forbidden_pattern {patterns}, numeric_range {json_path or regex, min, max},
   json_field_match {fields: {path: expected}, strict}, field_extraction {fields, strict}.
   Implementations may raise NotImplementedError until the run engine phase; only metadata and
   config validation are needed now.
4. Endpoints: GET /api/v1/eval/types -> [{type, label, description, definition_schema (JSON
   Schema)}]; GET /api/v1/eval/evaluators -> [{name, label, description, config_schema}].
   Read access for any principal.
5. Tests: each type validates good definitions and rejects bad ones (unknown operator, bad regex,
   missing value, unknown evaluator, bad evaluator config, extra fields); endpoints return all
   four types and all evaluators with schemas.
```

## Phase 3: Placement and scopes on the hierarchy

```text
PHASE 3 - PLACEMENT AND SCOPES
Inheritance semantics (write them in the HierarchyService module docstring):
- ASSIGNED to a node (playbook checks, skill bindings): applies to that node and all descendants.
- PLACED on a node (agents; later golden datasets and eval configs): receives what is assigned to
  its node and all ancestors.
- Users SCOPED to nodes: can see what is placed on those nodes and their descendants.

Agents:
1. Add agents.hierarchy_node_id (String 36, FK hierarchy_nodes.id, nullable, index, no cascade).
   Additive migration only (no data changes): "add hierarchy placement to agents".
2. Register: optional hierarchy_node_id (must exist and be active, else ValidationError
   "invalid_hierarchy_node"). If omitted, place on the single active node whose code equals the
   declared or overridden domain at level 2 (Line of business), case-insensitive; if zero or
   several match, leave unplaced.
3. agents.domain = code of the node's level-2 ancestor when placed at level 2 or 3; otherwise the
   declared domain stripped and lowercased. Remove the hardcoded domain list (CANONICAL_DOMAINS,
   coerce_domain, core/domains.py usages) and update affected tests.
4. PATCH accepts hierarchy_node_id (audited). GET /agents adds filters hierarchy_node_id and
   include_descendants (default true). Responses include hierarchy_node: NodeRef | None.
5. POST /api/v1/agents/placement/auto (HIERARCHY_ADMIN): places unplaced agents using rule 2,
   idempotent, returns {placed, skipped, details[]}. Used once after the hierarchy is created in
   Azure (existing Azure agents are unplaced after deployment).
6. register_node_reference("agents", ...).

Skills: if a Skill Fabric module exists, add skill_node_bindings (skill identifier,
hierarchy_node_id, unique pair), service bind/unbind/list, and visible skills for an agent =
direct agent grants UNION skills bound to the agent's node or any ancestor; endpoints
GET/PUT /api/v1/skill-fabric/{skill}/nodes (build.skills or HIERARCHY_ADMIN);
register_node_reference("skills", ...). If no skills module exists, skip and report.

Users: if no user table exists, add user_node_scopes (principal_id String(255),
hierarchy_node_id FK, unique pair), helper visible_node_ids(principal) -> set[str] | None
(None = unrestricted admin), and admin endpoints GET/PUT /api/v1/hierarchy/scopes/{principal_id}
(HIERARCHY_ADMIN). Do not enforce scopes elsewhere yet (RBAC will).
register_node_reference("user_scopes", ...).

Tests: placement by id, by level-2 code match, ambiguous/no match -> unplaced, invalid node 422,
domain derivation, list filter with descendants, auto-placement idempotency, node deletion
blocked by an agent/scope/binding, skills visibility via ancestor binding (if applicable).
```

## Phase 4: Eval Catalog checks

```text
PHASE 4 - EVAL CATALOG: CHECKS
1. Model EvalCheck (table eval_checks): key String(100) unique (pattern ^[a-z][a-z0-9_.-]{2,99}$,
   immutable), name String(200), description Text nullable, category_id FK eval_categories
   (required), subcategory_id FK eval_subcategories nullable (must belong to the category), type
   String(20) (EvalType), definition JSONType (validated with validate_definition), tags JSONType
   default [], default_weight Float (gt 0, le 1), pass_threshold Float (ge 0, le 1), hard_gate
   Boolean default false, status String(20) default draft, version Integer default 1, created_by,
   updated_by, timestamps.
2. Model EvalCheckVersion (table eval_check_versions): check_id FK, version, snapshot JSONType
   (all scoring fields + definition), created_by, created_at; UniqueConstraint(check_id, version).
   Write a version row on activation and whenever an ACTIVE check's definition, type,
   default_weight, pass_threshold or hard_gate changes (version + 1). Draft edits do not bump the
   version.
3. Rules: activation requires an active category (and active subcategory if set) ->
   ConflictError "category_not_active"; changing type requires a definition valid for the new
   type; archived checks cannot be edited (restore first); delete only draft and unreferenced
   -> "check_not_deletable". register a reference counter so categories/subcategories in use by
   checks cannot be deleted.
4. Responses: id, key, name, description, type, category {id, key, name}, subcategory
   {id, key, name} | null, definition, tags, default_weight, pass_threshold, hard_gate, status,
   version, assignment_count (0 until Phase 5), timestamps.
5. Endpoints (prefix /api/v1/eval/checks): GET list (filters category_id or category_key, type,
   status, tag, q; paginated), GET /{id}, GET /by-key/{key}, GET /{id}/versions, POST (201),
   PATCH /{id}, POST /{id}/status, DELETE /{id}. Writes require EVALS_AUTHOR; all audited.
6. Setup step "checks" (target "eval_catalog.checks"): complete when at least one active check
   exists.
7. Tests: create per type, invalid definitions 422, subcategory/category mismatch, activation
   rules, versioning (draft edit no bump, active edit bumps and writes snapshot), by-key,
   filters, permissions, deletion rules, category deletion blocked by a check.
```

## Phase 5: Eval Playbook

```text
PHASE 5 - EVAL PLAYBOOK (catalog checks mapped to the hierarchy)
1. Model EvalAssignment (table eval_assignments): node_id FK hierarchy_nodes, check_id FK
   eval_checks (no cascades), action String(20) in {add, override, disable}, weight_override,
   threshold_override (Float nullable, fractions), hard_gate_override (Boolean nullable),
   created_by, updated_by, timestamps; UniqueConstraint(node_id, check_id).
2. Resolver (app/services/eval_playbook.py), same semantics as the agent-eval EvalResolver:
   walk ancestors root -> target; at each node apply its assignments sorted by check key:
   add = include with the check's defaults plus any non-null overrides; override = merge non-null
   overrides into the already-resolved item (if not resolved yet, treat as add); disable = remove
   the item for this node and its descendants. Only ACTIVE checks resolve.
   Each resolved row: check_id, key, name, type, category {id, key, name}, weight,
   normalized_weight (within the resolved set), threshold, hard_gate, source ("direct" when the
   item's effective add/override happened at the target node, else "inherited"), source_node
   (NodeRef), overridden_at (NodeRef | None), own_assignment_id (the target node's assignment for
   this check, if any).
3. Endpoints (prefix /api/v1/eval/playbook):
   - GET /tree: active nodes nested with {id, code, name, level, level_label, children,
     resolved_count, direct_count}.
   - GET /nodes/{node_id}: {node (NodeRef), ancestors (NodeRef list), rows (grouped-ready list),
     stats {total, direct, inherited, hard_gates}, own_assignments}.
   - POST /nodes/{node_id}/assignments: body {check_id, action, overrides}. Validation: add only
     for active checks not already resolved at this node; override and disable only for checks
     resolved from an ancestor; overrides within 0-1. Errors: "check_already_resolved",
     "check_not_inherited", "check_not_active".
   - PATCH /assignments/{id} (overrides only), DELETE /assignments/{id}.
   Writes require EVALS_AUTHOR; audited.
4. Fill assignment_count on check responses (number of assignment rows referencing the check).
   register_node_reference("eval_assignments", ...) and a check reference counter (checks with
   assignments cannot be deleted).
5. Setup step "playbook" (target "eval_playbook"): complete when at least one active node has at
   least one resolved check.
6. Tests with Global -> Canada -> Banking -> Lending and a sibling US branch:
   assigned at Global appears at Lending as inherited from Global; added at Banking appears at
   Lending as inherited from Banking; override at Lending changes weight and sets overridden_at
   with source "direct"; disable at Banking removes it for Banking and Lending but not for US;
   draft/archived checks excluded; normalized weights sum to 1; stats correct; tree counts
   correct; validation errors; deleting an assigned check or node is blocked.
```

## Phase 6: Interim UI (frontend-local) and API contract docs

```text
PHASE 6 - FRONTEND-LOCAL INTERIM PAGES (for testing) + API CONTRACT DOCS
All new files start with "// Interim UI: to be replaced by the approved design." Plain Tailwind,
no new dependencies, all API calls in src/lib/api/*, backend error detail always shown. Do NOT
modify frontend/.
1. Sidebar: "Evaluate": Eval Catalog (/evaluation/catalog), Eval Playbook (/evaluation/playbook),
   Golden Datasets (disabled); "Certify": Readiness Runs (disabled); "Administration": Hierarchy
   (/admin/hierarchy). Setup banner on evaluation pages maps step targets to these routes.
2. Admin Hierarchy page: tree with expand/collapse, level badge + level_label, status pill,
   actions only when valid (Add child hidden at L3, Edit, Activate/Archive/Restore, Delete
   draft), native <dialog> forms (code on create only).
3. Reusable NodePicker (<select> of active nodes, indented "L2 · Global › Canada › Banking",
   props value/onChange/maxLevel/allowEmpty).
4. Agent Registry: NodePicker in the register dialog, placement path on cards ("Unplaced"),
   NodePicker filter including descendants, and an admin button "Auto-place agents" calling the
   placement endpoint and showing {placed, skipped}.
5. Eval Catalog tabs (?tab=): "Categories" (categories + sub-categories, weights shown as %) and
   "Checks": filters (category, type, status, tag, search); create/edit dialog where the
   definition form is generated from GET /eval/types and /eval/evaluators (rule builder rows;
   evaluator select + config fields; rubric + criteria; hybrid = surface + deep sections);
   weight and threshold entered as % in the UI and sent as fractions; tags input; status actions;
   version history panel.
6. Eval Playbook: left = playbook tree with resolved/direct counts; right = selected node:
   breadcrumb of ancestors, stats chips (total, direct, inherited, hard gates), table grouped by
   category with columns Eval name | Type | Weight (%) | Threshold (%) | Gate ("Must pass") |
   Source ("Direct" or "Inherited from <node>", plus "Overridden at <node>") | Actions (Add check
   from active checks not resolved; Override dialog; Disable here for inherited rows; Remove for
   direct rows).
7. Docs: docs/api/hierarchy.md and docs/api/eval.md with every endpoint, request/response
   examples, units (fractions 0-1), error codes, and a mapping table from the team UI's mock
   types (TEvalTemplate, TPlaybook, TPlaybookNode) to our responses, including frontend-side
   mappings: weight/threshold x100 for display, hard_gate -> "Must pass", validated ->
   status == "active", scope -> the node the check is assigned to, kind -> level_label,
   eval_count -> resolved_count.
8. Run npm run lint, typecheck and build in frontend-local.
```

## Phase 7: Verification

```text
PHASE 7 - VERIFICATION
1. Backend: full pytest; `alembic heads` shows one head; on SQLite: back up acp_td.db, downgrade
   to cbfebab50887, upgrade head, confirm the table list; restore the backup if needed.
2. SQL Server check without connecting: render offline DDL for the branch migrations
   (alembic upgrade cbfebab50887:head --sql with a temporary mssql URL in DATABASE_URL) and review
   it for SQL Server issues (cascade paths, index key lengths, JSON columns as NVARCHAR(max),
   booleans as BIT, CHECK constraints). Report findings; do not commit the generated SQL.
3. Frontend-local: lint, typecheck, build.
4. Final report: per phase summary, endpoints and tables added, migrations created, deviations
   from the spec and why, and follow-ups for the Skill Fabric owner, the RBAC owner, the
   teammate integrating frontend/, and infra (Azure SQL alembic stamp at cbfebab50887 before the
   first migration deploy).
```

---

## End-to-end check after Phase 6 (in frontend-local)

1. **Hierarchy:** create Global (L0) → Canada (L1) → Banking (L2) → Lending (L3), plus US (L1) → Banking (L2). Activate top-down.
2. **Categories:** create Accuracy, Safety, Compliance, Experience and Operational with their weights. Activate them.
3. **Checks:** create one **rule** check (e.g. "APR disclosure": response contains any of "APR" or "annual percentage rate", gate on) and one **prompt** check. Activate both.
4. **Playbook:**
   - Add the prompt check at **Global**. Lending should show it as *Inherited from Global*.
   - Add the rule check at **Banking (Canada)**. Lending shows it as inherited; US Banking does **not** show it.
   - Override the weight at **Lending**. The row should show *Direct* plus *Overridden at Lending*.
5. **Agents:** register a reference agent placed at Lending, and check the placement path on its card.

After Phase 7, the next PRs are **Golden Datasets** (upload to storage, validation report, versions) and then **Readiness Runs** (the scoring engine). I'll write those once these phases are done.
