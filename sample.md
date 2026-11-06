Phase 3 looks good. Both deviations are correct, and you can accept them.

1. **`Uuid` instead of `String(36)`:** Copilot is right, and my standing rule was wrong for this codebase. A foreign key column must have the same type as the primary key it points to. Here those primary keys are `sa.Uuid`, which is `UNIQUEIDENTIFIER` on SQL Server, so `NVARCHAR(36)` would be rejected.
2. **One migration instead of two:** fine. It's one phase, one additive revision, and it changes no data.

The autoflush fix also improves things, because a bad `hierarchy_node_id` is now rejected before anything is written.

Before you commit, ask Copilot one quick question. Skills live on the filesystem, so their bindings can't have a foreign key to anything.

```text
Quick check before commit, no new features:
1. When binding a skill to a node, do we validate the skill exists in the Skill Fabric
   (422 if not)? If not, add that check and a test.
2. If a bound skill is later removed from the filesystem, what do GET /skill-fabric/{skill}/nodes
   and the visible-skills calculation do? They should not error; orphaned bindings should be
   ignored in visibility and reported (e.g. "skill_missing": true) in the nodes listing.
   Add a test.
3. Confirm skill bind/unbind and scope PUT are written to the audit log.
Answer briefly, fix only what is missing, run pytest, then STOP.
```

Then commit as `feat(hierarchy): agent placement, skill bindings and user scopes`.

**Before Phase 4**, replace the "Databases" line in the standing rules with this:

```text
- Databases: production Azure SQL (SQL Server), local SQLite acp_td.db. Portable types only.
  IDs and foreign keys use the same type as the referenced primary key (sa.Uuid for
  hierarchy_nodes, eval_categories, eval_subcategories and all new eval tables); String(n) for
  natural keys with indexed strings <= 450 chars; Text, Integer, Float, Boolean,
  DateTime(timezone=True), JSONType. No ON DELETE CASCADE and no cascade relationships (enforce
  delete rules in services). No partial indexes or DB-specific defaults.
```

This matters in Phases 4 and 5, where `eval_checks.category_id`, `subcategory_id`, `eval_check_versions.check_id`, and `eval_assignments.node_id` and `check_id` must all be `Uuid`. Copilot would probably do this anyway, but with the rule in the standing rules it won't have to guess.
