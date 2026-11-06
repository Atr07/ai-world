Decisions on the frontend review (the backend API is the contract; presentation mapping is the
frontend's job):

A. Model/API changes to fold into the coming phases:
1. Hierarchy code pattern: widen to ^[a-z][a-z0-9_-]{1,49}$ (hyphens allowed, path separator
   stays "."); add a test.
2. Weights and thresholds are fractions 0-1 in the API and DB (default_weight gt 0 le 1,
   pass_threshold ge 0 le 1, normalized weights 0-1). Document this in docs/api and the schema
   descriptions. UIs multiply by 100 for display.
3. hard_gate stays boolean.
4. Phase 5 adds GET /api/v1/eval/playbook/tree: active hierarchy nodes with {id, code, name,
   level, level_label, children, resolved_count, direct_count}. The generic hierarchy tree
   endpoint stays free of eval data.
5. Hierarchy levels: still ONE root. L0 = Global (the organisation), L1 = Country (e.g. Canada,
   US), L2 = Line of business (e.g. Banking, Insurance), L3 = Business function (e.g. Lending).
   Update the level label constant to "Global", "Country", "Line of business",
   "Business function". The team UI's template "scope" (Global/Canada/US) maps to the hierarchy
   node a check is assigned to in the playbook; no separate jurisdiction field.
6. Add tags: list[str] (JSONType, default []) to eval checks with a tag filter on the list
   endpoint.
7. Check responses include both id and key; add GET /api/v1/eval/checks/by-key/{key}.
8. Check responses embed category {id, key, name} and subcategory {id, key, name} | null; list
   filters accept category_id or category_key.
9. Hierarchy node responses include level_label from the backend constant.
10. Setup steps return a "target" key (hierarchy, eval_catalog.categories, eval_catalog.checks,
    eval_playbook) instead of a URL path; frontend-local maps targets to its own routes. Update
    existing steps and tests.
11. Check responses include assignment_count; the playbook node response includes
    stats {total, direct, inherited, hard_gates}.

B. Frontend: the team's React SPA (frontend/) is the real UI and will be integrated by the
   teammate; do NOT modify frontend/. Keep building the interim pages in frontend-local as
   planned in Phase 6, because we test with it. Additionally write docs/api/eval.md and
   docs/api/hierarchy.md: every endpoint with request/response examples, field units
   (fractions), error codes, and a mapping table from the team's mock types (TEvalTemplate,
   TPlaybook, TPlaybookNode) to our responses, including the frontend-side mappings (percent
   display, "Must pass" label for hard_gate, validated = status active, scope = assigned node).

Apply A.1, A.5, A.9 and A.10 now, run tests.

