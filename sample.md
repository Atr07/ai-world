PHASE 2b - ERROR DETAILS AND LAYERING (small follow-up to Phase 2)
1. Structured validation errors:
   - Add an optional `errors: list[dict] | None` to AppError (default None). Each item uses
     FastAPI's shape: {"loc": [...], "msg": str, "type": str}.
   - Add UnprocessableError(AppError) with status 422. Keep BadRequestError (400) for business
     rules such as max_depth_exceeded.
   - Update the error handler. The response stays {"detail", "code"} and adds "errors" only when
     the list is present, so existing responses don't change.
   - invalid_definition raises UnprocessableError with detail "Invalid <type> definition." and
     errors converted from the Pydantic ValidationError (loc prefixed with "definition", e.g.
     ["definition","conditions",0,"value"]).
   - Update the Phase 2 tests to assert 422, the code, and at least one errors[].loc. Add a test
     showing that an existing 400/404/409 response has no "errors" key.
2. Layering fix:
   - Move evaluator metadata (names, labels, descriptions, config Pydantic models, lookup
     helpers) from app/services/evaluators/registry.py to app/core/evaluators.py, next to
     app/core/eval_types.py.
   - app/schemas/eval_definitions.py imports only from app/core.
   - app/services/evaluators/ keeps only implementations (still NotImplementedError for now).
     Implementations are looked up by evaluator name, and app/core never imports from services.
   - Update the imports, the GET /api/v1/eval/evaluators route and the tests.
3. Keep combine default "all" and surface_weight default 0.3. Document both defaults in the
   schema docstrings and the JSON Schema descriptions.
Run full pytest, show alembic heads (should be unchanged - no migration in this phase), list
changed files, then STOP.
Commit as "refactor(eval): structured validation errors and evaluator metadata in core".

