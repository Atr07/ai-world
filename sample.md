Phase 4 is close, but there's one gap in versioning, and the migration bug needs a permanent fix rather than a manual one. All three of Copilot's decisions are fine.

**Decisions**

1. **`invalid_category` / `invalid_subcategory` as 422:** accept. This matches `invalid_hierarchy_node`.
2. **`check_archived` (409):** accept.
3. **Tag filter in Python:** accept for now, since catalog size is small. Just make sure `total` counts the rows *after* the tag filter, or the pagination will be wrong. If the catalog grows a lot, a `eval_check_tags` table is the later fix.

**Versioning gap.** Take this path: activate (v1) → archive → restore (back to draft) → edit the definition → activate again. The guard sees that v1 already exists and writes nothing, so the latest snapshot no longer matches what will actually be scored. Runs record the check version they used, so this must be exact.

**The JSONB bug.** Copilot fixed it by hand. That depends on someone remembering to check every future migration, and the reference that render comes from was Postgres anyway. Two changes make this impossible to miss: tell Alembic how to render `JSONType`, and run migrations in the test suite.

Paste this as Phase 4b:

```text
PHASE 4b - VERSIONING FIX AND MIGRATION SAFETY (no new features)
1. Activation versioning: on activation, compare the check's current scoring fields (type,
   normalized definition, default_weight, pass_threshold, hard_gate) with its latest
   EvalCheckVersion snapshot.
   - No snapshot yet -> write version 1.
   - Snapshot identical -> no new row, version unchanged.
   - Snapshot different (e.g. edited while in draft after a restore) -> version += 1 and write
     the snapshot.
   Test: activate -> archive -> restore -> edit definition -> activate gives version 2 with the
   new definition; the same path without the edit stays at version 1 with one snapshot row.
2. Tag filter: confirm Page.total is the count after the tag filter; add a test with tags on
   page boundaries.
3. Alembic rendering of JSONType:
   - Add a render_item hook in alembic/env.py (context.configure, both online and offline) so
     autogenerate renders JSONType as `JSONType()` and adds
     `from app.db.types import JSONType` (use the real module path) to the migration imports.
   - If JSONType has a PostgreSQL JSONB variant, tell me whether anything still needs it (our
     targets are SQLite and SQL Server). Don't remove it yet; just report.
   - Do NOT edit migrations on origin/main. In branch-only migrations, leave the existing sa.Text()
     fix as is unless switching to JSONType() is a no-op for the DDL; report what you did.
4. Migration test: add tests/test_migrations.py that, on a temporary SQLite file, runs
   alembic upgrade head, then downgrade to cbfebab50887, then upgrade head again, using the
   Alembic command API. Also assert there is exactly one head. Mark it so it runs in the normal
   pytest run (it must be fast); it must not touch acp_td.db.
Run full pytest, show alembic heads, list changed files, then STOP.
Commit as "fix(eval): version on reactivation and test migrations in CI".
```

Commit Phase 4 itself first, as `feat(eval): eval catalog checks with versioning`, then run 4b as its own commit. After that, Phase 5 is unchanged.
