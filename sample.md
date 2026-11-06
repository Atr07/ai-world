PHASE G5 - CHECK USAGE LOOKUP AND OWNER REF (small)
1. GET /api/v1/eval/checks/{id}/assignments -> list of {assignment_id, action, node (NodeRef),
   weight_override, threshold_override, hard_gate_override, created_at}, ordered by node path.
   One query with a join; no tree walking. Read access for any principal.
2. GET /api/v1/eval/checks/{id}/usage -> {assigned_nodes: [NodeRef], resolved_node_count,
   disabled_at: [NodeRef]}. resolved_node_count = active nodes where the check currently
   resolves; compute it by resolving the active tree once in memory (load nodes + assignments in
   two queries), not one call per node. Document the cost and that it's fine at hierarchy scale.
3. GoldenDatasetOwnerRef: add domain. Don't change NodeRef (refs stay compact by design; clients
   use GET /hierarchy/tree for descriptions).
4. frontend-local: in the Eval Catalog check detail/edit view, a "Used in" panel listing
   assigned nodes with their action, and "Applies to N nodes".
5. Update docs/api/eval.md (new endpoints, plus a "Team UI field mapping" note: scope = check
   assignments, validated = status active) and docs/api/golden-datasets.md (location = hierarchy
   placement, source = version source).
6. Tests: assignments list after add/override/disable at different nodes; usage counts with
   inheritance and disable; empty for unassigned checks; owner ref includes domain.
Run full pytest, frontend-local lint/typecheck/build, list changed files, then STOP. No migration.
Commit as "feat(eval): check usage lookup and owner agent domain".


