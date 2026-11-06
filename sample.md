Keeping the same revision ID for a **branch-only, never-deployed** migration is acceptable, and the chain stays linear. Two adjustments, though, before you run anything.

## 1. Don't use the downgrade command it gave you. Recreate the local database instead

Copilot's `downgrade 3c01e38f4f19` will most likely **fail partway**. A downgrade runs each migration's `downgrade()` function, and migration `05ef25fc4478` now contains the **new** code: it drops `eval_subcategories` and `eval_categories` (new meaning). Your database still has the **old** tables (`eval_dimensions`, and `eval_categories` with the old meaning). So the rollback tries to drop `eval_subcategories`, which doesn't exist, and errors out, leaving the database half rolled back.

Since nothing is seeded and the local database only holds branch work, the clean option is a fresh database file. You'll lose locally entered data only: taxonomy, hierarchy and any locally registered agents, all of which take minutes to re-enter in the portal.

## 2. Answer Copilot's cascade question: yes, fix it now

Fold the cascade fix into the branch-only migration **before** you rebuild the database, so you only rebuild once. Paste this to Copilot:

```text
Yes, drop ondelete="CASCADE" on eval_subcategories.category_id now, in the same branch-only
migration 05ef25fc4478 and in the model (deletion rules stay enforced in the service: a category
can only be deleted when it is draft and has no subcategories, which already exists). While doing
it, audit EVERY foreign key in all branch-only migrations and models (audit_logs, eval taxonomy,
hierarchy_nodes) and remove any ondelete="CASCADE" or cascade="all, delete-orphan" relationship
settings that would create cascade paths on SQL Server; list what you changed. Do not touch
migrations that exist on origin/main. Run the full test suite afterwards, show `alembic heads` and
`alembic history`, and stop.
```

## 3. Then rebuild the local database

Run from `backend`, after Copilot finishes the step above:

```powershell
cd clu_agenticPlane\backend

# 1. Confirm the chain: 3c01e38f4f19 must be the last revision that is also on main
.\.venv\Scripts\alembic.exe heads
.\.venv\Scripts\alembic.exe history

# 2. Back up the old database, then remove it
Copy-Item acp_td.db acp_td.db.bak
Remove-Item acp_td.db

# 3. Build a fresh database from all migrations
.\.venv\Scripts\alembic.exe upgrade head
.\.venv\Scripts\alembic.exe current

# 4. Confirm the tables
.\.venv\Scripts\python.exe -c "import sqlite3; c=sqlite3.connect('acp_td.db'); print([r[0] for r in c.execute(\"select name from sqlite_master where type='table' order by name\")])"

# 5. Tests
.\.venv\Scripts\python.exe -m pytest
```

**Expected:**

- `heads` shows one head (`6695cce96994`).
- `history` shows `3c01e38f4f19` (main) then `05ef25fc4478` then `6695cce96994`, plus the E0 audit migration in between if it's branch-only.
- The table list includes `eval_categories`, `eval_subcategories` and `hierarchy_nodes`, plus the agent/skill tables from main's migrations. There should be **no** `eval_dimensions`.

**Check before step 2:** in `alembic history`, make sure `3c01e38f4f19` is the revision Azure SQL is on (the `version_num` your teammate saw). If it isn't, stop and tell me what you see.

## 4. Re-enter your data in the portal

1. **Administration → Hierarchy:** Bank (L0), then Banking (L1), then Lending (L2). Activate top-down.
2. **Eval Catalog → Categories:** Accuracy, Safety, Compliance, Experience, Operational with your weights, then activate.
3. Re-register your reference agents if you had any locally.

Once it all checks out, delete `acp_td.db.bak` 
