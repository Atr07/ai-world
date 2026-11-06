PHASE 6c - CONDITIONAL RULES AND HYBRID SURFACE THRESHOLD (schema only; execution comes later)
1. RuleCondition: add optional applies_when {field: str, values: list[str] (min 1)}. field refers
   to a test-case metadata field (e.g. "product_type"). A condition with applies_when is only
   evaluated when the case's metadata[field] is in values (case-insensitive); otherwise it is
   skipped, not failed. If every condition is skipped, the rule result is "not_applicable".
   Document this in the schema description and JSON Schema.
2. HybridDefinition: add optional surface_threshold (0-1). If set and the surface score is below
   it, the check fails without running deep_eval. Default null (always run both). Keep
   surface_weight.
3. CodeDefinition: add invert_score: bool = false (score = 1 - score), for evaluators where a match
   means failure.
4. All additions are optional with defaults, so existing checks stay valid. Changing these
   fields on an ACTIVE check counts as a definition change (version bump, as today).
5. frontend-local: rule builder row gets an optional "Only when" section (metadata field + values);
   hybrid form gets "Fail fast below surface score (%)"; code form gets an "Invert score" toggle.
6. Update docs/api/eval.md. No migration (definition is JSON). Run full pytest and frontend-local
   lint/typecheck/build, list changed files, then STOP.
Commit as "feat(eval): conditional rule conditions, hybrid surface threshold, invert score".
