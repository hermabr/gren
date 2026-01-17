Progress for Migration Feature

Context
- Goal: add explicit migration/alias support for Gren objects (rename/move/field changes), add gren_version default 1.0, block implicit recompute for active aliases, log migrations/overwrites, and show migration state in dashboard.
- Terminology:
  - aliased object = new object created by migration (alias directory).
  - original object = pre-migration object (existing directory).
- Explicit migrations only: no implicit legacy lookup. Alias directories must be created via gren.migrate or dashboard-triggered migration.
- State format: alias dir has metadata.json + state.json with result.status="migrated" and no SUCCESS.json. migrated_at lives only in migration.json.
- Migration record: migration.json exists in both dirs for alias/move; copy leaves original unchanged except for events. Record includes from/to namespace+hash+version+root, policy, origin, note, migrated_at.
- New behavior: alias is only active while migration.json has overwritten_at == null and original is success. If original is not success, the alias is detached and runs its own _create.

Decisions (current)
- Always alias by default; move/copy optional.
- gren_version is the only version field (no separate current version field). Default 1.0, always stored in metadata, omitted from to_python if default.
- Explicit migrations only: no implicit resolution or legacy lookup.
- _create must not run for active migrated aliases unless force recompute. If original is not success, alias recompute is allowed and detaches the alias.
- Migration events should be logged in events.jsonl for both original and aliased dirs.
- Overwrite logging uses a distinct event: migration_overwrite.
- Dashboard is display-only for now (no migration endpoints yet), but should be ready for future auto/manual migrations.
- Aliased object view: show aliased metadata/config, but also surface original status info if available and provide a button to view the original.

Detailed implementation plan
1) State model cleanup
   - src/gren/storage/state.py: remove migrated_at from _StateResultMigrated and _coerce_result; state.json stores only result.status="migrated".
   - src/gren/migrate.py: _write_migrated_state should only set result.status="migrated" and clear attempt.

2) Migration record detachment
   - src/gren/storage/migration.py: add overwritten_at: str | None = None to MigrationRecord.
   - Update MigrationManager read/write accordingly.

3) Alias activation + resolution logic
   - src/gren/core/gren.py:
     - Alias is active only when (state.result.status == "migrated") AND (migration.overwritten_at is None) AND (original result is success).
     - gren_dir resolves to original only for active aliases; otherwise stays on alias dir.
     - _is_migrated_state returns True only for active aliases so GrenMigrationRequired only blocks _create in that case.

4) Alias detachment + overwrite logging
   - In load_or_create, when force recompute is enabled or original is not success:
     - Compute in alias dir.
     - Mark overwritten_at in migration.json for both dirs.
     - Append migration_overwrite event to events.jsonl in both dirs with payload: policy, from, to, reason.

5) Dashboard API (resolved/original views)
   - src/gren/dashboard/api/models.py: add migration fields (kind, policy, migrated_at, overwritten_at, origin, note, from/to identifiers, original_result_status).
   - src/gren/dashboard/scanner.py: support view=resolved|original. Resolved view uses aliased metadata/config but includes original status if available.
   - src/gren/dashboard/api/routes.py: add a view query param for list/detail endpoints.

6) Frontend display
   - After make frontend-generate, update list + detail views to show migrated tag and toggle resolved/original.
   - Detail view should show aliased config by default, with a button to view original.
   - Update dashboard-frontend/src/api.test.ts mock data for migration fields.

7) Tests + verification
   - Add tests for migration behavior (alias active vs detached, overwrite logging).
   - Add dashboard tests for resolved/original view and migrated tags.
   - Update CHANGELOG.md.
   - Run make lint and make test (plus dashboard tests if modified).

Completed work
- [x] Added migration registry and helpers
  - New file: src/gren/serialization/migrations.py
    - MigrationContext, FieldRename, FieldAdd (with default_factory), Transform, MigrationSpec, MigrationRegistry, MIGRATION_REGISTRY.
    - Registry supports chaining and logs a warning if chain length > 1.
  - Updated: src/gren/serialization/__init__.py exports MigrationSpec/FieldAdd/FieldRename/Transform/MIGRATION_REGISTRY and DEFAULT_GREN_VERSION.
- [x] Serializer changes
  - src/gren/serialization/serializer.py
    - Added DEFAULT_GREN_VERSION = 1.0.
    - Added _is_default_value() that checks chz defaults/default_factory for a field.
    - to_python() omits gren_version if it matches default.
- [x] Migration metadata
  - New file: src/gren/storage/migration.py
    - MigrationRecord includes kind (alias/migrated), policy (alias/move/copy), from/to namespace+hash+version+root, migrated_at, origin, note.
    - MigrationManager handles read/write of migration.json and resolving directories.
  - src/gren/storage/__init__.py exports MigrationManager, MigrationRecord.
- [x] Errors
  - src/gren/errors.py: Added GrenMigrationRequired (inherits GrenError).
  - src/gren/__init__.py updated to export GrenMigrationRequired and migrate function.
- [x] Core changes (partial)
  - src/gren/core/gren.py
    - Added gren_version: float = chz.field(default=1.0) to Gren class.
    - Added helper methods: _is_migrated_state(), _migration_target_dir().
    - exists() now resolves alias target and returns True if target is success.
    - load_or_create() blocks when migrated unless force recompute (check is in place after reconcile).
    - gren_dir now resolves alias if migration.json kind=alias.
    - Added StateOwner usage for submitit queued attempt creation (some typing ignores still present).
- [x] Migration API (partial)
  - New file: src/gren/migrate.py
    - migrate(from_obj, to_obj, policy=alias|move|copy, origin, note).
    - Writes migration.json in new dir; and old dir for alias/move (not for copy).
    - Writes events in both old and new dirs.
    - For alias: writes state.result="migrated" in new dir.
    - For move/copy: copies payload and state, then writes metadata for new object.
    - uses DEFAULT_GREN_VERSION for target version check.

Current issues / blockers
- [ ] src/gren/storage/state.py still expects migrated_at in _StateResultMigrated; this must be removed.
- [ ] migrate.py _write_migrated_state still writes migrated_at; should only set status.
- [ ] No overwritten_at in migration.json yet (needed for alias detachment tracking).
- [ ] Alias activation logic does not yet check original success or overwritten_at.
- [ ] No migration_overwrite logging yet.
- [ ] Dashboard API/FE migration display still missing.

Files modified/added so far
- [x] Added: src/gren/serialization/migrations.py
- [x] Added: src/gren/storage/migration.py
- [x] Added: src/gren/migrate.py
- [x] Modified: src/gren/serialization/serializer.py
- [x] Modified: src/gren/serialization/__init__.py
- [x] Modified: src/gren/storage/metadata.py
- [x] Modified: src/gren/errors.py
- [x] Modified: src/gren/core/gren.py
- [ ] Modified: src/gren/storage/state.py (needs cleanup)
- [x] Modified: src/gren/__init__.py
