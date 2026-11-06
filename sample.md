Here's a single prompt you can paste into **GitHub Copilot Chat in Agent mode**. It tells Copilot to inspect your current code first, report back, and then implement everything in phases with checkpoints.

**What I verified in agent-eval before writing it:**

- `use_case_nodes` with L0–L3 and a dot path.
- `EvalResolver`: root-to-leaf `add / override / disable`, where an override with no prior add counts as an add. Each resolved item carries `source_node` and `is_inherited`, which is exactly your Direct/Inherit column.
- Template `category` uses your 5 values and `eval_type` your 4 types.
- Users are scoped only by domain, and skills have no hierarchy. Both get added here.
- It also covers your new facts: **Azure SQL (SQL Server)** in production, SQLite (`acp_td.db`) locally, and eval tables existing only locally.

**How to use it:**

1. Create the branch first: `git checkout -b feature/hierarchy-and-playbook`.
2. Paste the prompt. Copilot stops after **Phase 0** (the inspection report).
3. Check the report, reply "proceed with Phase 1", and so on. Don't let it run every phase in one go.

```text
ROLE AND WORKING RULES
You are working in this monorepo (backend/ FastAPI + SQLAlchemy 2.0 async + Alembic,
frontend/ Next.js App Router + TypeScript + Tailwind, agents/ AEI reference agents).
Implement a platform-wide hierarchy (L0-L3) with inheritance, align the eval taxonomy, and build
the Eval Catalog checks and Eval Playbook. Work in PHASES. Before writing any code, do Phase 0
(inspect and report) and STOP for my confirmation. After each later phase: run the tests, show
`alembic heads`, summarise the files changed, and STOP for my confirmation before the next phase.
Never seed data: every hierarchy node, category, check and assignment is created through the API
and portal. Follow the existing code conventions you find (routers in app/api/v1, logic in
app/services, AppError subclasses mapped to {"detail","code"}, pydantic-settings, record_audit,
LifecycleStatus/assert_transition, require_capability/PrincipalDep, Page/PaginationParams,
JSONType, pytest + httpx AsyncClient with in-memory SQLite fixtures). Keep routes thin. Type hints
and docstrings everywhere. No print().

TARGET ENVIRONMENT AND DATABASE RULES
- Production: Azure App Service + Azure SQL Database (Microsoft SQL Server). Local: SQLite file
  (acp_td.db). Code and migrations must work on BOTH.
- Azure SQL currently has only the agent and skill tables (from main). Eval tables exist only
  locally on this branch.
- SQL Server compatibility rules for every new model/migration:
  * Use portable types only: String(n) (keep indexed strings <= 450 chars), Text, Integer, Float,
    Boolean, DateTime(timezone=True), Uuid/String(36), JSONType (never raw JSONB, never ARRAY).
  * No partial/filtered indexes, no database-specific server defaults, no CHECK constraints that
    use non-portable functions (simple BETWEEN is fine).
  * Self-referencing and multi-path foreign keys: NO ON DELETE CASCADE (SQL Server rejects
    cycles/multiple cascade paths). Enforce delete rules in the service layer instead.
  * Remember SQL Server treats NULLs as equal in unique constraints; design constraints so this
    is correct (e.g. only one root can have parent_id NULL, which is intended).
- Migration rules: NEVER edit or delete migrations that exist on origin/main. New migrations must
  chain from the current head (exactly one head at all times). Migrations that exist only on this
  branch (check with `git diff --name-status origin/main -- backend/migrations/versions`) may be
  regenerated if a phase requires renaming tables; if so, first downgrade locally to the last
  main revision, delete only those branch-only files, then autogenerate. Review every generated
  migration by hand.

PHASE 0 - INSPECT AND REPORT (no code changes)
Report, as tables:
1. Alembic: `alembic heads`, `alembic current`, `alembic history`; which migration files are
   branch-only vs on origin/main; the latest revision that is on main.
2. Existing models and tables; whether the eval taxonomy currently uses "dimension/category" or
   "category/subcategory" naming; whether eval checks/templates, assignments, golden datasets,
   eval configs or runs already exist.
3. Agent registry: how `domain` is set (look for CANONICAL_DOMAINS / coerce_domain /
   core/domains.py), the register/patch/list endpoints and their tests.
4. Skill Fabric: does a skills module/table exist (visibility per agent)? Users/RBAC: does any
   user or scope table exist, or only the permission stub?
5. Permission stub and capability constants; setup-status step registry; storage/LLM/task-runner
   interfaces from E0.
6. Frontend: sidebar structure, existing evaluation pages/tabs, shared components (dialogs,
   pickers, api client, getErrorMessage).
Then list the exact changes you plan per phase below, flag any conflicts, and STOP.

PHASE 1 - HIERARCHY CORE (platform-wide)
Model HierarchyNode (table hierarchy_nodes): id, code String(50) (schema pattern
^[a-z][a-z0-9_]{1,49}$), name String(120), description String(1000) nullable, level Integer
(CheckConstraint level BETWEEN 0 AND 3), parent_id FK hierarchy_nodes.id nullable index (no
cascade), path String(255) unique index (codes joined by "."), sort_order Integer default 0,
status String(20) index default "draft", timestamps; UniqueConstraint(parent_id, code).
Rules (HierarchyService in app/services/hierarchy.py):
- No parent -> L0; only ONE L0 node may exist (ConflictError "root_exists"). With parent ->
  level = parent.level + 1; level > 3 -> ValidationError "max_depth_exceeded"; parent must exist
  and not be archived; path = parent.path + "." + code; duplicate code under same parent ->
  ConflictError "node_code_exists". Branches may stop at any level (L3 is never mandatory).
- code, parent_id and level are immutable after create (no move in this phase).
- Lifecycle via assert_transition: activate requires an active parent (except root) ->
  "parent_not_active"; archive requires no active children -> "node_has_active_children";
  delete only draft, no children, no references -> "node_not_deletable".
- Reference registry: register_node_reference(name, counter(session, node_id) -> int) and
  count_references(node_id); every module that links to nodes registers a counter.
- Query helpers used by ALL modules: get_tree(status=None), list_nodes(level, status, parent_id),
  get_ancestors(node_id, include_self=True) ordered root first, get_descendant_ids(node_id,
  include_self=True) using path equality OR path LIKE path + '.%' (must not match sibling
  prefixes like "bank" vs "banking"), is_within(node_id, scope_node_id), to_ref(node) -> NodeRef
  {id, code, path, name, level}.
- record_audit on every create/update/status/delete (entity_type "hierarchy_node").
API (prefix /api/v1/hierarchy): GET /tree, GET /nodes (filters), GET /nodes/{id},
GET /nodes/{id}/ancestors, GET /nodes/{id}/descendants, GET /nodes/{id}/references,
POST /nodes, PATCH /nodes/{id} (name, description, sort_order), POST /nodes/{id}/status,
DELETE /nodes/{id}. Add capability HIERARCHY_ADMIN = "hierarchy.admin" (local admin has it);
writes require it.
Setup step "hierarchy" registered FIRST: complete when an active L0 and at least one active L1
exist; link "/admin/hierarchy".
Tests: root/levels/paths, second root 409, depth 4 rejected, duplicate code rules, partial
branches valid, lifecycle rules, ancestors/descendants (including the prefix case), reference
counter blocks delete, 403 without capability, setup step.

PHASE 2 - EVAL TAXONOMY ALIGNMENT
Target naming: Category (top level, has default_weight, e.g. Accuracy, Safety, Compliance,
Experience, Operational, all entered by an admin in the portal, never seeded) and optional
Sub-category under a Category. If the current code uses "dimension/category", rename to
"category/subcategory" everywhere (models, tables eval_categories/eval_subcategories, schemas,
service, routes /api/v1/eval/taxonomy/categories and /subcategories, error codes, audit entity
types, setup step text, tests, frontend types/api/labels). Apply the migration rules above.
Categories are global (not per hierarchy node). Keep lifecycle and normalized weights over active
categories.
Eval TYPES are NOT a table: define StrEnum EvalType {prompt, code, rule, hybrid} in code and expose
GET /api/v1/eval/types returning each type with label, description and the JSON schema of its
definition (see Phase 4) so the portal can render the right form.

PHASE 3 - PLACEMENT AND SCOPES ON THE HIERARCHY
Inheritance semantics (document them in a module docstring in app/services/hierarchy.py):
- Items ASSIGNED to a node (eval checks via playbook, skill bindings) apply to that node and all
  descendants.
- Entities PLACED on a node (agents, later golden datasets and eval configs) RECEIVE everything
  assigned to their node and all ancestors.
- Users SCOPED to nodes can see entities placed on those nodes and their descendants.
Agents:
- Add agents.hierarchy_node_id (FK, nullable, index, no cascade). Register request: optional
  hierarchy_node_id (must exist and be active, else ValidationError "invalid_hierarchy_node");
  if omitted, place on the active L1 node whose code equals the declared/overridden domain
  (case-insensitive), else leave unplaced.
- agents.domain = code of the node's L1 ancestor when placed; otherwise the declared domain
  stripped and lowercased. Remove the hardcoded domain list (CANONICAL_DOMAINS, coerce_domain,
  core/domains.py usages) and update affected tests.
- PATCH accepts hierarchy_node_id (audited). GET /agents adds filters hierarchy_node_id and
  include_descendants (default true). Responses include hierarchy_node: NodeRef | None.
- Register node reference counter "agents".
Skills: if a Skill Fabric module exists, add a table skill_node_bindings (skill_name or skill_id,
hierarchy_node_id, unique pair) plus service helpers bind/unbind/list and
visible_skills_for_agent = direct agent grants UNION skills bound to the agent's node or any
ancestor; register counter "skills". If no skills module exists, skip and report it.
Users: if no user table exists, add user_node_scopes (principal_id String(255), hierarchy_node_id
FK, unique pair) and a helper visible_node_ids(principal) -> set[str] | None (None = admin,
unrestricted) built from descendants of each scope node; admin-only endpoints GET/PUT
/api/v1/hierarchy/scopes/{principal_id}. Do NOT enforce scopes on other endpoints yet (RBAC
module will); register counter "user_scopes".
Tests for each.

PHASE 4 - EVAL CATALOG: CHECKS
Model EvalCheck (table eval_checks): key String(100) unique (pattern ^[a-z][a-z0-9_.-]{2,99}$,
immutable), name String(200), description Text nullable, category_id FK eval_categories
(required, category must be active when the check is activated), subcategory_id FK nullable (must
belong to the category), type String(20) (EvalType), definition JSONType, default_weight Float
(gt 0, le 1), pass_threshold Float (0..1), hard_gate Boolean default false, status (lifecycle),
version Integer default 1, timestamps.
Definition validation per type (Pydantic discriminated by type; reject unknown fields):
- rule: {conditions: [{field: "response" | "expected" | "json:<path>", operator: one of
  contains, not_contains, contains_any, contains_all, equals, not_equals, regex, not_regex,
  number_between, length_between, json_field_equals, value(s) as needed}], combine: "all"|"any"}.
- code: {evaluator: one of the registered deterministic evaluator names (exact_match, regex,
  keyword_presence, forbidden_pattern, numeric_range, json_field_match, field_extraction),
  config: object validated by that evaluator's schema}. Provide a small evaluator registry module
  exposing each evaluator's name, label and config JSON schema (implementations may be stubs that
  raise NotImplementedError until the run engine phase).
- prompt: {rubric: str (min 20 chars), criteria: list[str] optional, score_scale: {min, max}}.
- hybrid: {pre_check: a rule or code definition, judge: a prompt definition, pre_check_weight:
  0..1}.
Editing an ACTIVE check's definition/weight/threshold/gate increments version (keep a JSONType
history list or an eval_check_versions table; choose one and justify). Archived checks cannot be
assigned. Delete only draft and unreferenced.
API (prefix /api/v1/eval/checks): list (filters category_id, type, status, q; paginated), get,
create, patch, status change, delete; plus GET /api/v1/eval/types and
GET /api/v1/eval/evaluators. Writes require EVALS_AUTHOR. Setup step "checks": at least one
active check. Register counter on categories so a category in use cannot be deleted.
Tests for validation per type, versioning, lifecycle, permissions.

PHASE 5 - EVAL PLAYBOOK (map catalog checks to the hierarchy)
Model EvalAssignment (table eval_assignments): node_id FK hierarchy_nodes (no cascade), check_id
FK eval_checks (no cascade), action String(20) in {add, override, disable}, weight_override,
threshold_override, hard_gate_override (nullable), created_by, timestamps;
UniqueConstraint(node_id, check_id).
Resolver (app/services/eval_playbook.py), same semantics as the agent-eval reference:
walk ancestors root -> target; at each level apply that node's assignments sorted by check key:
add = include with the check's defaults plus any overrides; override = merge non-null overrides
into an already-resolved item (if none exists, treat as add); disable = remove from the resolved
set. Only ACTIVE checks are resolved. Each resolved row returns: check_id, key, name, type,
category {id, name}, weight, threshold, hard_gate, source ("direct" if the item's effective add or
override happened at the target node, else "inherited"), source_node: NodeRef, overridden_at:
NodeRef | None, assignment_id of the target node's own assignment if any.
Also return normalized weights within the resolved set.
API (prefix /api/v1/eval/playbook): GET /nodes/{node_id} (resolved rows + the node's own
assignments), POST /nodes/{node_id}/assignments (add/override/disable with validation: override
and disable only for checks resolved from an ancestor; add only for checks not already resolved),
PATCH /assignments/{id}, DELETE /assignments/{id}. Writes require EVALS_AUTHOR; all changes
audited. Register node reference counter "eval_assignments" and check reference counter.
Setup step "playbook": at least one active node has at least one resolved check.
Tests: inheritance Bank -> Banking -> Lending (assigned at Banking appears on Lending as
inherited from Banking); override at Lending changes weight and marks overridden_at; disable at
Lending removes it only for Lending's subtree; sibling Insurance unaffected; archived/draft checks
excluded; validation errors.

PHASE 6 - FRONTEND (interim UI, plain Tailwind, no new dependencies; put
"// Interim UI: to be replaced by the approved design." at the top of each new file)
- Sidebar groups: "Evaluate": Eval Catalog (/evaluation/catalog), Eval Playbook
  (/evaluation/playbook), Golden Datasets (disabled); "Certify": Readiness Runs (disabled);
  "Administration": Hierarchy (/admin/hierarchy). Keep the existing setup banner on evaluation
  pages (hidden when all steps are complete).
- Admin Hierarchy page: tree with expand/collapse, level badges L0-L3, status pills, actions
  shown only when valid (Add child hidden at L3, Edit, Activate/Archive/Restore, Delete draft),
  native <dialog> forms, backend error detail shown.
- Reusable NodePicker (<select> of active nodes, indented "L2 · Bank › Banking › Lending",
  props value/onChange/maxLevel/allowEmpty).
- Agent Registry: NodePicker in the register dialog; placement path on cards ("Unplaced" when
  none); NodePicker filter (includes descendants).
- Eval Catalog page tabs (?tab=): "Categories" (categories + sub-categories) and "Checks": list
  with filters (category, type, status, search); create/edit dialog whose definition form is
  generated from GET /eval/types schema for the chosen type (rule builder rows for rule; evaluator
  select + config fields for code; rubric textarea for prompt; both for hybrid); status actions;
  version shown.
- Eval Playbook page: left = hierarchy tree (active nodes), right = table for the selected node
  with columns Eval name | Type | Category | Weight | Threshold | Gate (Must pass) | Source
  ("Direct" or "Inherited from <node name>", plus "Overridden at <node>" when set) | Actions
  (Add check: picker of active checks not already resolved; Override: dialog for
  weight/threshold/gate; Disable here: for inherited rows; Remove: for direct rows). Show
  normalized weight %.

PHASE 7 - VERIFICATION
- Backend: full pytest; `alembic heads` shows one head; on SQLite run `alembic downgrade` to the
  last main revision and `alembic upgrade head` successfully.
- SQL Server check without connecting: generate offline DDL for the branch migrations
  (alembic upgrade <last_main_revision>:head --sql with a temporary mssql URL in DATABASE_URL)
  and review it for SQL Server issues (cascade cycles, index lengths, JSON columns, booleans).
  Report findings; do not commit the generated SQL.
- Frontend: npm run lint, typecheck, build.
- Final report: summary per phase, endpoints added, tables added, migrations created, any
  deviations from this spec and why, and follow-ups for Skill Fabric and RBAC owners.
```

**After Phase 0**, check these three things in Copilot's report before saying "proceed":

1. The latest revision on `main` (the one Azure SQL is on) is correctly identified.
2. All eval migrations are listed as **branch-only**.
3. It correctly detected whether a skills module and any user tables exist.

If any of those look wrong, describe what the report says and I'll adjust the instructions for that phase.
