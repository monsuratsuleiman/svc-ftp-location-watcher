# ftp-watcher Capability — Delta

## ADDED Requirements

### Requirement: DiscoveryFactory provides default and advanced construction paths

The library SHALL expose `DiscoveryFactory` as the public construction
entry point for the discovery subsystem, with two paths:

- `DiscoveryFactory.create(config: WatcherConfig): DiscoveryRuntime` —
  default; wires all discovery components from config alone.
- `DiscoveryFactory.builder(config: WatcherConfig): DiscoveryRuntimeBuilder` —
  advanced; allows component overrides before building.

Both paths apply the discovery validation profile (defined in change 2)
at construction time. A misconfigured config fails to construct; the
runtime never starts.

#### Scenario: Default factory wires production components

- **GIVEN** a `WatcherConfig` valid under the discovery profile
- **WHEN** `DiscoveryFactory.create(config)` is called
- **THEN** it returns a `DiscoveryRuntime` with all components wired
  from config
- **AND** the runtime is ready to `start()` without further setup

#### Scenario: Builder allows component overrides

- **GIVEN** a `WatcherConfig` valid under the discovery profile
- **WHEN** `DiscoveryFactory.builder(config).withSecretGetter(stub).build()`
  is called
- **THEN** the resulting `DiscoveryRuntime` uses the stub `SecretGetter`
- **AND** all other components are wired from config as in the default
  path

#### Scenario: Both paths reject invalid configurations equivalently

- **GIVEN** a `WatcherConfig` invalid under the discovery profile
  (e.g. missing `transport` block)
- **WHEN** either `DiscoveryFactory.create(config)` or
  `DiscoveryFactory.builder(config).build()` is called
- **THEN** both fail with the same validation errors
- **AND** neither produces a `DiscoveryRuntime`

#### Scenario: Discovery profile tolerates absent pipeline-only fields

- **GIVEN** a `WatcherConfig` with `db.pipeline` absent and locations
  missing pipeline-only fields (`destination`, `topic`, `onFailure`)
- **WHEN** `DiscoveryFactory.create(config)` is called
- **THEN** construction succeeds
- **AND** the returned `DiscoveryRuntime` is valid

### Requirement: DiscoveryRuntime exposes lifecycle methods

`DiscoveryRuntime` SHALL be a public type with:

- `start()` — non-blocking. Performs synchronous startup (DB migration
  apply, connection pool open, Kafka client creation), then launches
  background coroutines. Throws on synchronous startup failure.
- `stop()` — graceful shutdown within the configured grace period.
  Idempotent (repeated calls are no-ops).
- `isRunning(): Boolean` — current state.

#### Scenario: Start and stop lifecycle

- **GIVEN** a constructed `DiscoveryRuntime`
- **WHEN** `start()` is called
- **THEN** `isRunning()` returns true
- **AND** synchronous startup work (migration apply, pool open) has
  completed before `start()` returns
- **WHEN** `stop()` is called
- **THEN** in-flight work completes or is cancelled within the grace
  period
- **AND** `isRunning()` returns false

#### Scenario: Synchronous startup failure throws

- **GIVEN** a `DiscoveryRuntime` whose configured database is
  unreachable
- **WHEN** `start()` is called
- **THEN** it throws an exception describing the failure
- **AND** `isRunning()` returns false
- **AND** subsequent `stop()` is a no-op

#### Scenario: Double start is idempotent or throws

- **GIVEN** a `DiscoveryRuntime` already started
- **WHEN** `start()` is called again
- **THEN** the call either is a no-op or throws `IllegalStateException`
- **AND** no duplicate background coroutines are launched

#### Scenario: Double stop is a no-op

- **GIVEN** a `DiscoveryRuntime` already stopped
- **WHEN** `stop()` is called again
- **THEN** the call returns without error
- **AND** no exception is thrown

### Requirement: DiscoveryRuntimeBuilder provides the documented override surface

`DiscoveryRuntimeBuilder` SHALL allow overriding the following components
via fluent `with*` methods:

- `withSecretGetter(SecretGetter): DiscoveryRuntimeBuilder`
- `withClock(Clock): DiscoveryRuntimeBuilder`
- `withDomainEventRouter(DomainEventRouter): DiscoveryRuntimeBuilder`

`build(): DiscoveryRuntime` SHALL apply the overrides and return the
runtime. Validation runs at `build()` time, equivalent to
`DiscoveryFactory.create()`.

Additional overrides (`withTransport`, etc.) are added by later changes
alongside the abstractions they swap.

#### Scenario: SecretGetter override is honoured

- **GIVEN** a stub `SecretGetter`
- **WHEN** `DiscoveryFactory.builder(config).withSecretGetter(stub).build()`
- **THEN** the returned runtime uses the stub for all secret resolution

#### Scenario: Clock override is honoured

- **GIVEN** a fixed `Clock`
- **WHEN** `DiscoveryFactory.builder(config).withClock(fixed).build()`
- **THEN** the returned runtime uses the fixed clock for all
  time-stamped operations (e.g. `discovery_files.discovered_at`)

#### Scenario: DomainEventRouter override is honoured

- **GIVEN** an `InMemoryDomainEventRouter` test substitute
- **WHEN** `DiscoveryFactory.builder(config).withDomainEventRouter(inMemory).build()`
- **THEN** the returned runtime publishes to the in-memory router
- **AND** no Kafka broker is required

#### Scenario: Builder fluent chain returns same builder type

- **GIVEN** a fresh builder
- **WHEN** multiple `with*` calls are chained
- **THEN** each returns a `DiscoveryRuntimeBuilder` reference
- **AND** the final `build()` produces a runtime reflecting all overrides

### Requirement: PipelineFactory provides default and advanced construction paths

The library SHALL expose `PipelineFactory` as the public construction
entry point for the pipeline subsystem, mirroring `DiscoveryFactory`:

- `PipelineFactory.create(config: WatcherConfig): PipelineRuntime`
- `PipelineFactory.builder(config: WatcherConfig): PipelineRuntimeBuilder`

Both paths apply the pipeline validation profile (defined in change 2)
plus the topic-uniqueness rules added in this change.

#### Scenario: Default factory wires production components

- **GIVEN** a `WatcherConfig` valid under the pipeline profile
- **WHEN** `PipelineFactory.create(config)` is called
- **THEN** it returns a `PipelineRuntime` ready to `start()`

#### Scenario: Builder allows component overrides

- **GIVEN** a `WatcherConfig` valid under the pipeline profile
- **WHEN** `PipelineFactory.builder(config).withClock(fixed).build()`
- **THEN** the resulting runtime uses the fixed clock
- **AND** other components are wired from config

#### Scenario: Pipeline profile tolerates absent discovery-only fields

- **GIVEN** a `WatcherConfig` with `transport` absent and locations
  missing discovery-only fields (`source`, `pollIntervalSeconds`,
  `filenamePattern`)
- **WHEN** `PipelineFactory.create(config)` is called
- **THEN** construction succeeds

### Requirement: PipelineRuntime exposes lifecycle methods

`PipelineRuntime` SHALL provide the same lifecycle contract as
`DiscoveryRuntime`: non-blocking `start()` with synchronous-startup
throw semantics, idempotent `stop()`, accurate `isRunning()`.

#### Scenario: Start and stop lifecycle

- **GIVEN** a constructed `PipelineRuntime`
- **WHEN** `start()` is called
- **THEN** synchronous startup completes (migration apply, pool open,
  consumer subscription)
- **AND** `isRunning()` returns true
- **WHEN** `stop()` is called
- **THEN** `isRunning()` returns false within the grace period

#### Scenario: Synchronous startup failure throws

- **GIVEN** a `PipelineRuntime` whose Kafka broker is unreachable
- **WHEN** `start()` is called
- **THEN** it throws
- **AND** `isRunning()` returns false

### Requirement: PipelineRuntimeBuilder provides the documented override surface

`PipelineRuntimeBuilder` SHALL allow overriding:

- `withSecretGetter(SecretGetter)`
- `withClock(Clock)`
- `withDomainEventRouter(DomainEventRouter)`

`withDestination` is added by a later change. `withTransport` is not
exposed — pipeline does not use the source transport.

#### Scenario: PipelineRuntimeBuilder does not expose withTransport

- **GIVEN** the `PipelineRuntimeBuilder` public type
- **WHEN** its methods are inspected
- **THEN** no `withTransport` method is present
- **AND** the documented design rationale (pipeline reads from GCS, not
  from the source) is reflected in the absent override

### Requirement: Topic-uniqueness validation enforced by PipelineFactory

`PipelineFactory` SHALL apply two additional validation rules beyond the
pipeline profile in both the `create()` and `builder().build()` paths:

1. No two locations within the same `WatcherConfig` declare the same
   external `topic` value.
2. No location's external `topic` equals `internalTopicName`.

Violations fail construction with aggregated errors naming the offending
topic(s) and location(s).

These rules SHALL NOT be applied by `DiscoveryFactory` (discovery does
not read external `topic` values).

#### Scenario: Distinct external topics across locations

- **GIVEN** a config where every location's `topic` is unique
- **WHEN** `PipelineFactory.create(config)` is called
- **THEN** validation passes

#### Scenario: Two locations declaring the same external topic

- **GIVEN** a config where two locations declare
  `topic = "shared.received"`
- **WHEN** `PipelineFactory.create(config)` is called
- **THEN** construction fails
- **AND** the error names `"shared.received"` and the location IDs
  sharing it

#### Scenario: Location topic equals internalTopicName

- **GIVEN** a config where `internalTopicName = "watcher.events"` and
  some location declares `topic = "watcher.events"`
- **WHEN** `PipelineFactory.create(config)` is called
- **THEN** construction fails
- **AND** the error names the offending location and explains the
  clash with `internalTopicName`

#### Scenario: DiscoveryFactory ignores external topics

- **GIVEN** a config that would fail the pipeline topic-uniqueness
  rules
- **WHEN** `DiscoveryFactory.create(config)` is called
- **THEN** construction succeeds (assuming the discovery profile
  is otherwise satisfied)
- **AND** the topic-uniqueness rules are not applied

### Requirement: Public API surface is small and explicit

The library's public API SHALL be limited to the documented set listed
in this change's `design.md` ("Public API surface"). All other production
types SHALL be marked `internal`.

A build-time test SHALL enumerate the expected public surface and assert
that the library exposes exactly those types. Adding a new public type
without updating the expected list SHALL fail the build.

#### Scenario: Documented public types are present

- **GIVEN** the compiled library jar
- **WHEN** the public API surface is inspected
- **THEN** every type in the documented public surface is publicly visible

#### Scenario: Undocumented types are not public

- **GIVEN** the compiled library jar
- **WHEN** the public API surface is inspected
- **THEN** no type outside the documented surface is publicly visible

#### Scenario: Adding a new public type without updating the list fails the build

- **GIVEN** a developer makes an internal helper class public without
  updating `EXPECTED_PUBLIC_API`
- **WHEN** the build runs the API-surface test
- **THEN** the test fails
- **AND** the error names the type that drifted

## MODIFIED Requirements

### Requirement: Library exposes discovery and pipeline as separately bootstrappable subsystems

The library SHALL expose two factory entry points — `DiscoveryFactory`
and `PipelineFactory` — each producing a runtime for its subsystem.
Consuming services choose their topology by which factory or factories
they call. The library does not provide a `WatcherFactory` or composite
runtime type.

#### Scenario: Discovery-only consumer calls only DiscoveryFactory

- **GIVEN** a consumer service intended to run only discovery
- **WHEN** the consumer's `main()` runs
- **THEN** it calls `DiscoveryFactory.create(config).start()` (and
  registers a shutdown hook calling `stop()`)
- **AND** it does not reference `PipelineFactory`

#### Scenario: Pipeline-only consumer calls only PipelineFactory

- **GIVEN** a consumer service intended to run only pipeline
- **WHEN** the consumer's `main()` runs
- **THEN** it calls `PipelineFactory.create(config).start()`
- **AND** it does not reference `DiscoveryFactory`

#### Scenario: Composite consumer calls both factories from main

- **GIVEN** a consumer service intended to run both subsystems
- **WHEN** the consumer's `main()` runs
- **THEN** it constructs and starts both `DiscoveryRuntime` and
  `PipelineRuntime`
- **AND** the shutdown hook stops discovery before pipeline (so the
  publisher drains before the consumer shuts down)

### Requirement: Each factory validates only the slice of config it reads

The library SHALL apply validation at factory construction (both
`create` and `.builder().build()` paths):

- `DiscoveryFactory` applies the discovery profile.
- `PipelineFactory` applies the pipeline profile plus the
  topic-uniqueness rules added in this change.

The composite profile defined in change 2 SHALL be removed. Composite
topology consumers receive equivalent coverage from calling both
factories.

#### Scenario: DiscoveryFactory.create runs the discovery validator

- **GIVEN** any `WatcherConfig` passed to `DiscoveryFactory.create`
- **WHEN** the call evaluates
- **THEN** the discovery validation profile is applied
- **AND** errors are aggregated into a single failure if any

#### Scenario: PipelineFactory.create runs the pipeline validator and topic rules

- **GIVEN** any `WatcherConfig` passed to `PipelineFactory.create`
- **WHEN** the call evaluates
- **THEN** the pipeline validation profile is applied
- **AND** the two topic-uniqueness rules are applied
- **AND** all errors are aggregated into a single failure

#### Scenario: Composite validation profile is removed

- **GIVEN** the change-2 codebase that exposed a composite profile
- **WHEN** change 3 is applied
- **THEN** the composite profile is no longer exposed or callable
- **AND** any references in tests or code are removed

### Requirement: testFixtures provides three LocalRunner factory functions

The library's `testFixtures` source set SHALL expose three factory
functions on `LocalRunner`, each constructing runtimes via the public
factories:

- `LocalRunner.discoveryOnly(configPath)` — builds via
  `DiscoveryFactory.builder(config)`, applies test overrides, starts.
- `LocalRunner.pipelineOnly(configPath)` — builds via
  `PipelineFactory.builder(config)`.
- `LocalRunner.both(configPath)` — builds both, manages combined
  lifecycle, stops discovery before pipeline on shutdown.

#### Scenario: LocalRunner uses public factories

- **GIVEN** the `LocalRunner` source
- **WHEN** its construction paths are inspected
- **THEN** they call `DiscoveryFactory.builder()` and/or
  `PipelineFactory.builder()`
- **AND** they do not reach into any `internal` library types

#### Scenario: LocalRunner.both stops discovery before pipeline

- **GIVEN** `LocalRunner.both` is running both subsystems
- **WHEN** the runner is closed
- **THEN** `DiscoveryRuntime.stop()` is called first
- **AND** `PipelineRuntime.stop()` is called after discovery has stopped
- **AND** both complete within the grace period
