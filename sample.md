PHASE 4d - SQL SERVER-REVIEWABLE MIGRATIONS (no new features, branch-only migrations only)
1. In branch-only migrations (3c01e38f4f19 onward; never 0baefba90221 or cbfebab50887):
   - New tables and their indexes: replace op.batch_alter_table blocks with plain
     op.create_table / op.create_index (and op.drop_index / op.drop_table in downgrade).
   - Changes to EXISTING tables (e.g. agents.hierarchy_node_id): use a dialect check -
     SQLite -> batch_alter_table (it can't ALTER constraints); every other dialect -> plain
     op.add_column + op.create_index + op.create_foreign_key, and the reverse in downgrade.
     In offline mode, use context.get_context().dialect.name for the check (there's no
     connection to inspect).
2. Constraint names: every FK, unique constraint and index in branch migrations must have an
   explicit, deterministic name (e.g. fk_<table>_<column>_<reftable>, uq_<table>_<cols>,
   ix_<table>_<cols>), max 128 chars. Tell me whether Base.metadata has a naming_convention. If it
   doesn't, propose one (don't apply it yet) and report what `alembic check` would show if it
   were applied.
3. alembic/env.py: set render_as_batch to False so future autogenerate emits plain operations.
   Add a comment explaining the dialect-check pattern for changes to existing tables.
4. Verify:
   - alembic upgrade cbfebab50887:head --sql with a temporary mssql URL now completes. Review it
     for: no CASCADE, named constraints, NVARCHAR(max) for JSON, BIT for booleans,
     UNIQUEIDENTIFIER for Uuid, index key size within limits. Report findings; don't commit the
     SQL.
   - The same offline render for downgrade (head:cbfebab50887 --sql) also completes.
   - alembic check: no differences.
   - Full pytest including tests/test_migrations.py (SQLite round trip still works).
Show alembic heads, list changed files, then STOP.

