1. Levels (labels only, not enforced types; any branch may stop at any level):
   L0 = Organisation, L1 = Line of business, L2 = Business function, L3 = Use case.
   Example path: bank / banking / lending / mortgage_origination
   (names: Bank > Banking > Lending > Mortgage Origination).
   Store level as 0-3 on the node; expose the four labels from a backend constant so the UI
   shows "L2 · Business function".

2. One tree, single root. Exactly one L0 node. Enforce in the service (ConflictError
   "root_exists") and cover it with a test; do NOT rely on a unique constraint on NULL parent_id
   (SQL Server and SQLite treat NULLs differently).

3. Inheritance = add / override / disable, root to leaf (same as the agent-eval resolver):
   - add: the child includes a check not already inherited.
   - override: the child changes weight / threshold / hard_gate of an inherited check; non-null
     override fields MERGE into the inherited values, other fields stay inherited.
   - disable: the child suppresses an inherited check for itself and its descendants only.
   A child's override or disable never affects siblings or ancestors.

4. Option (c), rename only, and the taxonomy stays global (no node scope, no folding into
   hierarchy nodes): dimension -> category (top level, has the weight: Accuracy, Safety,
   Compliance, Experience, Operational, entered via the portal), category -> subcategory
   (optional). The migration is branch-only and not deployed, so regenerating it is acceptable:
   downgrade locally to the last main revision first, regenerate, and update the tests and portal
   labels. Do the rename as its own commit before the hierarchy work.

5. Check shape, corrected: key (immutable), name, description, category_id (required),
   subcategory_id (optional), type in {prompt, code, rule, hybrid} (EvalType enum in code, not a
   table), definition (JSON validated per type exactly as in Phase 4: rule = conditions + combine;
   code = registered evaluator + config; prompt = rubric + criteria + score_scale; hybrid =
   pre_check + judge + pre_check_weight), default_weight, pass_threshold, hard_gate, lifecycle
   status, version.

6. Playbook shape, corrected: NOT a named or versioned bundle. The playbook is the set of
   EvalAssignment rows (node_id, check_id, action add/override/disable, optional overrides).
   "The playbook of a node" = the resolved set from root to that node, with source
   direct/inherited, source_node and overridden_at. Runs are a later phase: a run will resolve the
   agent's placement node and snapshot the resolved checks at run time; no playbook versioning now.

7. Yes, agents get hierarchy_node_id (nullable FK). Keep the domain column (derived: the code of
   the L1 ancestor when placed, otherwise the declared domain lowercased). The migration must be
   additive only (add nullable column + index + FK, no data changes), safe for Azure SQL.
   Backfill: none in the migration, because hierarchy nodes do not exist in Azure yet. Existing
   agents stay unplaced with their domain unchanged. Add an idempotent admin endpoint
   POST /api/v1/agents/placement/auto that places unplaced agents on the active L1 node whose code
   matches their domain and returns {placed, skipped}; an admin runs it after creating the
   hierarchy in the portal.

8. Introduce a dedicated capability HIERARCHY_ADMIN = "hierarchy.admin" (the hierarchy is
   platform-wide, used by agents, skills, users and evals, so not EVALS_ADMIN). The local admin
   principal gets it.

9. Yes, fix the Principal capabilities spelling now, in its own commit, updating every reference
   and test.
