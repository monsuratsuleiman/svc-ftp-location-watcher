# Tasks — Restructure as Library

## 1. Resolve open questions

- [ ] 1.1 Ask the user for the library Maven coordinates. Update `project.md`.
- [ ] 1.2 Ask the user for the library versioning scheme. Update `project.md`.
- [ ] 1.3 Confirm `db-toolkit` supports library-packaged classpath
      migrations from subdirectories (`discovery/db/migrations/`,
      `pipeline/db/migrations/`). If not, design a workaround.
- [ ] 1.4 Confirm `service-bootstrap` can be consumed by a library that
      contributes a readiness check without owning service lifecycle.
- [ ] 1.5 Confirm the platform's routable-module API for HTTP routes.
- [ ] 1.6 Confirm `testFixtures` artifact publishing on the org's Artifactory.
- [ ] 1.7 Confirm `DomainEventRouter` Maven coordinates, current API,
      and shutdown lifecycle behaviour. Catalogue in `project.md`. Pin
      version in `gradle/libs.versions.toml`.
- [ ] 1.8 Confirm NotificationPublisher tick interval and per-tick batch
      cap defaults (proposed: 60s tick, 100 rows per tick per location).
      Document in the example HOCON.

## 2. Module restructuring

- [ ] 2.1 Rename Gradle module: `svc-` → `lib-`. Update
      `settings.gradle.kts` and Maven coordinates.
- [ ] 2.2 Enable the `java-test-fixtures` plugin in `build.gradle.kts`.
- [ ] 2.3 Remove production `main()` from `src/main`. Verify no
      production class declares a `main()`.
- [ ] 2.4 Mark all production types `internal` by default. (Full
      public-API design lands in change 3; for now, nothing not already
      public becomes public.)

## 3. Package layout

- [ ] 3.1 Create `shared/`, `discovery/`, `pipeline/` packages per the
      structure in `design.md`.
- [ ] 3.2 Move `WatcherConfig` and nested config types into
      `shared/config/`.
- [ ] 3.3 Define `FileDiscoveredEvent` in `shared/events/`.
- [ ] 3.4 Move (or create stubs for) `FileTransport` interface in
      `shared/transport/` (real implementation lands in change 4).
- [ ] 3.5 Set up `discovery/internal/` and `pipeline/internal/` packages
      for subsystem-internal wiring.
- [ ] 3.6 Add build-time enforcement that `discovery/` and `pipeline/`
      do not import each other (Detekt rule, or a simple test that
      scans import statements).

## 4. Per-subsystem state tables

- [ ] 4.1 Author `discovery/db/migrations/V001__create_discovery_files.sql`
      per the schema in `design.md`, including the partial index.
- [ ] 4.2 Author `pipeline/db/migrations/V001__create_ftp_file_state.sql`
      per the schema in `design.md`, keyed on `bundle_id`. Replaces the
      `ftp_file_state` migration shipped in change 1.
- [ ] 4.3 Wire discovery's runtime initialisation to apply discovery
      migrations against the configured `ftp.db.discovery` connection.
- [ ] 4.4 Wire pipeline's runtime initialisation to apply pipeline
      migrations against the configured `ftp.db.pipeline` connection.
- [ ] 4.5 Implement `DiscoveryStateStore` (CRUD on `discovery_files`,
      including the `ON CONFLICT DO NOTHING` insert and the
      `SELECT ... WHERE status='DISCOVERED' ORDER BY discovered_at`
      query). Note: PollLoop and NotificationPublisher *use* this in
      change 5; the store exists in this change.
- [ ] 4.6 Implement `PipelineStateStore` (CRUD on `ftp_file_state`,
      including the `ON CONFLICT (bundle_id) DO NOTHING` insert and
      transition helpers). Used by pipeline workers in change 5.

## 5. Per-factory validation rules

- [ ] 5.1 Implement `ConfigValidator.validateForDiscovery(config)`.
- [ ] 5.2 Implement `ConfigValidator.validateForPipeline(config)`.
- [ ] 5.3 Implement `ConfigValidator.validateForComposite(config)`.
- [ ] 5.4 Each validator aggregates errors and reports them in one pass.
- [ ] 5.5 Validators run only the slice of rules appropriate to the
      profile (per the spec's "Each factory validates only the slice"
      requirement).

## 6. Status routes split

- [ ] 6.1 Convert the change-1 status route into pipeline's routable
      module (`PipelineStatusRoutes`). Payload is a stub at this change;
      change 5 populates worker state.
- [ ] 6.2 Implement discovery's routable status module
      (`DiscoveryStatusRoutes`) covering: configured locations, per-
      location last poll timestamp, last successful publish timestamp,
      oldest unpublished `discovered_at`, `DISCOVERED` row count.
- [ ] 6.3 Verify neither module mounts itself; mounting is the
      consumer's responsibility.

## 7. Readiness check

- [ ] 7.1 Verify the change-1 readiness check still works. The library
      contributes the predicate; `service-bootstrap` hosts the endpoint
      when wired by a consumer or by `LocalRunner`.
- [ ] 7.2 The composite readiness predicate reports `READY` once both
      subsystem migrations have applied.
- [ ] 7.3 Discovery-only and pipeline-only deployments report `READY`
      once their own migration applies.

## 8. testFixtures

- [ ] 8.1 Create `src/testFixtures/kotlin/.../LocalRunner.kt` with three
      companion factory functions: `discoveryOnly`, `pipelineOnly`, `both`.
- [ ] 8.2 Create `LocalRunnerMain.kt` accepting `<mode> <configPath>`
      args; mode defaults to `both`.
- [ ] 8.3 Create `src/testFixtures/resources/localrunner.example.conf`
      per the example in `design.md`, annotated with HOCON comments.
- [ ] 8.4 Implement `StubSecretGetter` (Map-backed `SecretGetter`).
- [ ] 8.5 Implement `InMemoryDomainEventRouter` mirroring the platform
      `DomainEventRouter` API.
- [ ] 8.6 Add Gradle task `:runLocal` invoking `LocalRunnerMain`.

## 9. Documentation

- [ ] 9.1 Update README:
  - Project description: library, not service.
  - "Consuming this library" section: factory-per-mode pattern (sample
    consumer `main()` for each of `DiscoveryFactory`,
    `PipelineFactory`, `WatcherFactory` — the factories themselves land
    in change 3; this is the contract we're committing to).
  - "Topologies" section: composite vs split, when to choose each.
  - "Running locally" section: `LocalRunner` modes, `:runLocal` task,
    `localrunner.example.conf` walkthrough.
  - "Split deployment notes": **must keep `internalTopicName` and
    location IDs consistent across both services' HOCON files**.
  - Replace any remaining service-shape language with library shape.
- [ ] 9.2 Update `openspec/project.md`: catalogue `DomainEventRouter`;
      replace remaining `<TBD>` markers this change depends on.

## 10. Testing — regression (behavioural preservation)

- [ ] 10.1 Migrate change 1's unit tests into the library module.
      Adapt where necessary (e.g. validation tests now use composite
      profile by default).
- [ ] 10.2 Migrate change 1's integration tests into the library module.
      Adapt them to use `LocalRunner.both` where appropriate.
- [ ] 10.3 Run the full change-1 regression suite. Every change-1
      scenario passes. Any failure halts the change.

## 11. Testing — unit (new scenarios)

Test names track scenarios from `spec.md`. One test per scenario at
minimum; multiple tests per scenario where exhaustive coverage warrants.

- [ ] 11.1 `Library_jar_contains_no_production_main`
- [ ] 11.2 `TestFixtures_classes_not_in_main_classpath`
- [ ] 11.3 `Package_discovery_does_not_import_pipeline`
- [ ] 11.4 `Package_pipeline_does_not_import_discovery`
- [ ] 11.5 `DiscoveryStateStore_insert_is_idempotent_on_natural_key`
- [ ] 11.6 `DiscoveryStateStore_select_pending_orders_by_discovered_at`
- [ ] 11.7 `DiscoveryStateStore_status_check_constraint_rejects_invalid_values`
- [ ] 11.8 `PipelineStateStore_insert_is_idempotent_on_bundle_id`
- [ ] 11.9 `PipelineStateStore_status_check_constraint_excludes_DISCOVERED`
- [ ] 11.10 `FileDiscoveredEvent_serialises_with_schemaVersion_1_0`
- [ ] 11.11 `ConfigValidator_discovery_profile_accepts_no_pipeline_fields`
- [ ] 11.12 `ConfigValidator_pipeline_profile_accepts_no_discovery_fields`
- [ ] 11.13 `ConfigValidator_composite_profile_rejects_missing_pipeline_db`
- [ ] 11.14 `ConfigValidator_discovery_profile_rejects_missing_transport`
- [ ] 11.15 `ConfigValidator_aggregates_multiple_errors_in_one_pass`
- [ ] 11.16 `InMemoryDomainEventRouter_delivers_published_events_to_subscribers`

## 12. Testing — integration (new scenarios)

- [ ] 12.1 `LocalRunner_discoveryOnly_starts_only_discovery` —
      Testcontainers Postgres + Redpanda; assert only discovery's
      migration applied, only discovery coroutines running.
- [ ] 12.2 `LocalRunner_pipelineOnly_starts_only_pipeline` — assert
      only pipeline's migration applied, only pipeline coroutines
      running.
- [ ] 12.3 `LocalRunner_both_starts_both_subsystems` — assert both
      migrations applied, both subsystems running.
- [ ] 12.4 `Composite_with_shared_db_creates_both_tables_in_one_database`
- [ ] 12.5 `Composite_with_split_db_creates_each_table_in_its_own_database`
- [ ] 12.6 `Discovery_migration_idempotent_on_second_start`
- [ ] 12.7 `Pipeline_migration_idempotent_on_second_start`
- [ ] 12.8 `Discovery_status_routes_mountable_independently`
- [ ] 12.9 `Pipeline_status_routes_mountable_independently`

## 13. Pre-merge checklist

- [ ] 13.1 All `<TBD>` markers in `project.md` that this change depends
      on are resolved.
- [ ] 13.2 `DomainEventRouter` catalogued in `project.md` and pinned in
      `libs.versions.toml`.
- [ ] 13.3 Regression suite (section 10) passes.
- [ ] 13.4 Unit tests pass.
- [ ] 13.5 Integration tests pass.
- [ ] 13.6 `./gradlew check` passes.
- [ ] 13.7 Every new scenario in `spec.md` is covered by at least one
      named test (per `AGENTS.md`'s per-scenario gate).
- [ ] 13.8 No production `main()` in `src/main` (grep / build-time check).
- [ ] 13.9 testFixtures classes do not leak into the production jar
      (build-time assertion).
- [ ] 13.10 Build-time import-boundary check enforces `discovery/` ↔
      `pipeline/` non-coupling.
- [ ] 13.11 README updated with library shape, three-factory pattern,
      topology guidance, and split-deployment consistency note.
