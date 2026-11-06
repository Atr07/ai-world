PHASE 6b - SIMPLIFY SUB-CATEGORIES
Design decision: a sub-category is only an optional grouping label inside a category. It has no
weight. Check weights come from the playbook, and category weights drive the readiness score.
Sort order is display-only.

1. Remove weight from eval_subcategories: model, schemas, service validation, setup-step logic and
   tests. New additive migration "drop weight from eval_subcategories": SQLite uses the batch
   pattern, other dialects plain op.drop_column; downgrade re-adds the column as nullable.
   Don't edit earlier migrations. Must render offline for mssql.
2. Sub-categories stay optional everywhere:
   - A check's subcategory_id stays nullable, and creating or activating a check never requires one.
   - The categories setup step must not require any sub-category to exist.
   - Key unique within its category (keep uq_eval_subcategories_category_id_key), so the same key
     (e.g. "handoff") can exist under two categories.
3. sort_order on categories and sub-categories becomes optional in the API (default 0; list
   ordering = sort_order, then name). Hide it from the create forms in frontend-local; keep it in
   the edit dialog labelled "Display order".
4. Category weight stays required (fraction 0-1, shown as %). Helper text in the Categories tab:
   "How much this category counts toward the readiness score." If the active categories' weights
   don't add up to 100%, show a warning, not an error (scoring will normalize them).
5. Sub-category form: key, name, optional description only, under a heading
   "Sub-categories (optional grouping)". In the Checks dialog the Sub-category select shows
   "None" by default and lists only sub-categories of the selected category.
6. Checks tab: next to the Type select, show the selected type's description from
   GET /eval/types, so it's clear that category is the score bucket and type is how the check runs.
7. Update docs/api/eval.md (sub-category fields, optional sort_order, category weight meaning).
8. Verify: full pytest, tests/test_migrations.py, offline mssql render (upgrade and downgrade),
   alembic check (no differences), frontend-local lint/typecheck/build. Show alembic heads, list
   changed files, then STOP.
Commit as "refactor(eval): sub-categories as optional grouping labels, optional display order".

