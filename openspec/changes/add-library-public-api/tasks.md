# Tasks — Add Library Public API

## 1. Resolve open questions

- [ ] 1.1 Decide the build-time API-surface check approach (Kotlin
      metadata scanner, reflection, or Detekt rule). Default: Kotlin
      metadata scanner.
- [ ] 1.2 Confirm `DomainEventRouter` shutdown lifecycle: managed by
      `service-bootstrap` or closed explicitly by `Runtime.stop()`.
- [ ] 1.3 Confirm grace-period default for `stop()` (proposed 30s) is
      acceptable to platform conventions.
- [ ] 1.4 Confirm Kotlin coroutine scope ownership pattern matches
      org conventions (the runtime owns a `SupervisorJob` scope;
      `stop()` cancels it with a timeout).

## 2. Factories and runtimes

- [ ] 2.1 Implement `DiscoveryFactory` as a public object exposing
      `create(WatcherConfig)` and `builder(WatcherConfig)`.
- [ ] 2.2 Implement `DiscoveryRuntime` as a public class with
      `start()`, `stop()`, `isRunning()`.
- [ ] 2.3 Implement `DiscoveryRuntimeBuilder` with fluent `withSecretGetter`,
      `withClock`, `withDomainEventRouter`, and `build()`.
- [ ] 2.4 Route `DiscoveryFactory.create(config)` through
      `builder(config).build()` so both paths share validation and
      wiring.
- [ ] 2.5 Implement `PipelineFactory`, `PipelineRuntime`,
      `PipelineRuntimeBuilder` (parallel structure). Do NOT expose
      `withTransport`.
- [ ] 2.6 Implement an internal `*RuntimeAssembler` for each subsystem
      that takes config + overrides and produces a wired runtime. The
      assemblers are NOT public.

## 3. Lifecycle implementation

- [ ] 3.1 `start()` performs synchronous startup work in order: apply
      migrations, open DB pool, create Kafka client / consumer,
      contribute readiness check `READY`. Then launches background
      coroutines (empty at this change; populated in change 5).
- [ ] 3.2 Synchronous-startup failures throw; runtime stays "never
      started"; `isRunning()` returns false.
- [ ] 3.3 `stop()` is idempotent (compareAndSet on running flag).
      Cancels coroutine scope with `gracePeriod` timeout.
      Closes DB pool and (if appropriate per 1.2) Kafka client.
- [ ] 3.4 `start()` after `stop()` either no-ops or throws
      `IllegalStateException` ("runtime is single-use"). Document the
      decision; tests cover whichever shape is chosen.
- [ ] 3.5 Double `start()` either no-ops or throws `IllegalStateException`
      ("already started").

## 4. Validation wiring

- [ ] 4.1 `DiscoveryRuntimeBuilder.build()` calls
      `ConfigValidator.validateForDiscovery(config)`. Throws
      `ConfigValidationException` on any error.
- [ ] 4.2 `PipelineRuntimeBuilder.build()` calls
      `ConfigValidator.validateForPipeline(config)` and then applies
      the two topic-uniqueness rules. Aggregates all errors before
      throwing.
- [ ] 4.3 **Remove** the composite validation profile from change 2:
      `ConfigValidator.validateForComposite` is deleted, along with any
      references in tests or wiring.

## 5. Topic-uniqueness rules

- [ ] 5.1 Implement "no two locations share an external `topic`" check
      as a method on `PipelineRuntimeBuilder.build()` or in
      `ConfigValidator`. Error message lists the offending topic and
      sharing location IDs.
- [ ] 5.2 Implement "no location's `topic` equals `internalTopicName`"
      check. Error message names the offending location and explains
      the clash.
- [ ] 5.3 Verify `DiscoveryFactory.build()` does NOT apply these rules.

## 6. Public API surface enforcement

- [ ] 6.1 Define `EXPECTED_PUBLIC_API` constant in
      `PublicApiSurfaceTest` listing every documented public type.
- [ ] 6.2 Implement `scanLibraryPublicTypes()` per the chosen approach
      from task 1.1. Inspect compiled jar; exclude testFixtures and
      test classes.
- [ ] 6.3 Implement `PublicApiSurfaceTest.public_api_surface_matches_expected`
      asserting both directions: every expected type is public, no
      unexpected type is public.
- [ ] 6.4 Mark every type listed in `design.md`'s public surface as
      public; mark everything else `internal`.
- [ ] 6.5 Verify the test fails when a non-listed type is made public
      (e.g. by removing `internal` from an assembler).

## 7. LocalRunner migration

- [ ] 7.1 Refactor `LocalRunner.discoveryOnly` to use
      `DiscoveryFactory.builder(config).withSecretGetter(...).build()`.
- [ ] 7.2 Refactor `LocalRunner.pipelineOnly` to use
      `PipelineFactory.builder(config)...build()`.
- [ ] 7.3 Refactor `LocalRunner.both` to build both runtimes via their
      respective factories. Stops discovery before pipeline on close.
- [ ] 7.4 Verify LocalRunner does not reach into any `internal` library
      types (compile-time check via the API-surface test ensures this).

## 8. Documentation

- [ ] 8.1 Update README "Consuming this library" section:
  - Minimal discovery-only `main()` example.
  - Minimal pipeline-only `main()` example.
  - Minimal composite `main()` example (both factories, shutdown hook
    stops discovery first).
- [ ] 8.2 Update README "API stability" section: enumerate the public
      surface; document the versioning contract.
- [ ] 8.3 Update README "Topologies" section: composite vs split,
      lifecycle ordering recommendation.
- [ ] 8.4 Update sample HOCON: show fields used by each factory's
      validation profile. The same HOCON works for all three modes.

## 9. Testing — unit

- [ ] 9.1 `DiscoveryFactory_create_returns_runnable_runtime_for_valid_config`
- [ ] 9.2 `DiscoveryFactory_create_aggregates_validation_errors_for_invalid_config`
- [ ] 9.3 `DiscoveryFactory_create_tolerates_absent_pipeline_fields`
- [ ] 9.4 `DiscoveryFactory_builder_allows_secretGetter_override`
- [ ] 9.5 `DiscoveryFactory_builder_allows_clock_override`
- [ ] 9.6 `DiscoveryFactory_builder_allows_domainEventRouter_override`
- [ ] 9.7 `DiscoveryFactory_builder_rejects_invalid_config_at_build`
- [ ] 9.8 `DiscoveryFactory_create_and_builder_produce_equivalent_runtimes_for_same_config`
- [ ] 9.9 `DiscoveryRuntime_start_then_stop_clean_lifecycle`
- [ ] 9.10 `DiscoveryRuntime_synchronous_startup_failure_throws`
- [ ] 9.11 `DiscoveryRuntime_double_start_is_idempotent_or_throws`
- [ ] 9.12 `DiscoveryRuntime_double_stop_is_noop`
- [ ] 9.13 `DiscoveryRuntime_isRunning_reflects_state`
- [ ] 9.14 `DiscoveryRuntime_stop_completes_within_grace_period`
- [ ] 9.15 `PipelineFactory_create_returns_runnable_runtime_for_valid_config`
- [ ] 9.16 `PipelineFactory_create_tolerates_absent_discovery_fields`
- [ ] 9.17 `PipelineFactory_builder_allows_secretGetter_override`
- [ ] 9.18 `PipelineFactory_builder_allows_clock_override`
- [ ] 9.19 `PipelineFactory_builder_allows_domainEventRouter_override`
- [ ] 9.20 `PipelineFactory_create_and_builder_produce_equivalent_runtimes_for_same_config`
- [ ] 9.21 `PipelineRuntime_start_then_stop_clean_lifecycle`
- [ ] 9.22 `PipelineRuntime_synchronous_startup_failure_throws`
- [ ] 9.23 `PipelineRuntime_double_start_is_idempotent_or_throws`
- [ ] 9.24 `PipelineRuntime_double_stop_is_noop`
- [ ] 9.25 `PipelineRuntime_isRunning_reflects_state`
- [ ] 9.26 `PipelineRuntimeBuilder_does_not_expose_withTransport`
- [ ] 9.27 `PipelineFactory_rejects_duplicate_external_topics_across_locations`
- [ ] 9.28 `PipelineFactory_rejects_location_topic_matching_internalTopicName`
- [ ] 9.29 `DiscoveryFactory_ignores_external_topic_uniqueness_rules`
- [ ] 9.30 `Public_api_surface_matches_expected`
- [ ] 9.31 `Adding_undocumented_public_type_fails_surface_test`
- [ ] 9.32 `Composite_validation_profile_is_removed_from_codebase`

## 10. Testing — integration

- [ ] 10.1 `LocalRunner_uses_public_factories_for_construction` —
      reflection / source inspection confirms LocalRunner does not
      reach into `internal` types.
- [ ] 10.2 `LocalRunner_both_stops_discovery_before_pipeline` —
      observe stop-order via timestamps or instrumented runtimes.
- [ ] 10.3 `Consumer_main_pattern_starts_discovery_runtime` — small
      in-process consumer; Testcontainers Postgres + Redpanda; build
      via `DiscoveryFactory`, start, verify readiness READY, stop.
- [ ] 10.4 `Consumer_main_pattern_starts_pipeline_runtime` — analogous
      for pipeline.
- [ ] 10.5 `Consumer_main_pattern_starts_both_runtimes` — composite
      mode; build via both factories; verify both reach READY.

## 11. Pre-merge checklist

- [ ] 11.1 Unit tests pass.
- [ ] 11.2 Integration tests pass.
- [ ] 11.3 `./gradlew check` passes.
- [ ] 11.4 Every new scenario in `spec.md` is covered by at least one
      named test.
- [ ] 11.5 Public API surface test passes (only documented types are
      public).
- [ ] 11.6 Composite validation profile is removed (task 4.3 verified
      by grep / search of codebase).
- [ ] 11.7 README updated with consuming, stability, and topology
      sections.
- [ ] 11.8 LocalRunner uses only public factory APIs (no `internal`
      imports).
