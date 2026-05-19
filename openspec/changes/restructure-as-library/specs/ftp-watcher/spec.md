# ftp-watcher Capability — Delta

## ADDED Requirements

### Requirement: Watcher is delivered as a Gradle library

The watcher SHALL be published as a Gradle library artifact
(`lib-ftp-location-watcher`). It SHALL NOT include a `main()` entry point
in its production classpath. Local execution is available via
`LocalRunner` in the library's `testFixtures` source set.

#### Scenario: Library jar has no production main

- **GIVEN** the published library artifact
- **WHEN** its production classpath is inspected
- **THEN** no class declares a production `main()` function
- **AND** the artifact is consumable as a Gradle dependency

#### Scenario: testFixtures artifact does not leak into production classpath

- **GIVEN** a build that depends on `lib-ftp-location-watcher` (not its
  testFixtures variant)
- **WHEN** the production classpath is inspected
- **THEN** `LocalRunner`, the in-memory router, and other testFixtures
  classes are not present

### Requirement: Library exposes discovery and pipeline as separately bootstrappable subsystems

The library SHALL be organised into two production subsystems with
non-overlapping responsibilities:

- **Discovery subsystem** — polls transports and notifies the internal
  topic about newly-seen files.
- **Pipeline subsystem** — consumes notifications, downloads files,
  uploads them, and publishes external events.

Consuming services SHALL be able to bootstrap either subsystem alone, or
both together (composite topology), without code changes in the library.
Mode is determined by which factory the consuming service calls (codified
in change 3), not by configuration.

The two subsystems SHALL communicate exclusively via the internal Kafka
topic. They SHALL NOT call each other's code or read each other's state
tables.

#### Scenario: Discovery and pipeline are in separate Kotlin packages

- **GIVEN** the library source tree
- **WHEN** the package structure is inspected
- **THEN** discovery code lives under `…/discovery/`
- **AND** pipeline code lives under `…/pipeline/`
- **AND** types depended on by both live under `…/shared/`
- **AND** no class under `…/discovery/` imports from `…/pipeline/`
- **AND** no class under `…/pipeline/` imports from `…/discovery/`

#### Scenario: Discovery-only deployment runs without pipeline components

- **GIVEN** a consumer that bootstraps only the discovery subsystem
  (via `LocalRunner.discoveryOnly` in this change; via
  `DiscoveryFactory.create(config).start()` from change 3 onward)
- **WHEN** the runtime starts
- **THEN** discovery's PollLoop and NotificationPublisher are running
- **AND** no pipeline workers, no pipeline event consumer, no pipeline
  state-table connection pool is created

#### Scenario: Pipeline-only deployment runs without discovery components

- **GIVEN** a consumer that bootstraps only the pipeline subsystem
- **WHEN** the runtime starts
- **THEN** the pipeline event consumer is running
- **AND** no discovery PollLoop, no NotificationPublisher, no
  `discovery_files` connection pool is created

### Requirement: Discovery has two cooperating loops per location

Discovery SHALL run two cooperating coroutines per configured location:

- **PollLoop** — runs at the location's `pollIntervalSeconds`. On each
  tick, calls `transport.list(location)` and INSERTs newly-seen files
  into `discovery_files` with `status = DISCOVERED`, `bundle_id = uuid`,
  and `discovered_at = now()`. Uses
  `ON CONFLICT (location_id, filename, mtime) DO NOTHING`. PollLoop
  SHALL NOT publish to Kafka.
- **NotificationPublisher** — runs at the configured tick interval
  (default 60 seconds). On each tick, queries `discovery_files` for the
  location's `DISCOVERED` rows ordered by `discovered_at ASC`, publishes
  `FileDiscoveredEvent` for each (serially, awaiting broker ack before
  the next), and on each successful ack transitions the row to
  `NOTIFIED` with `notified_at = now()`. Caps work per tick at the
  configured batch size (default 100). On publish failure, the row
  remains `DISCOVERED` and the next tick retries.

NotificationPublisher SHALL be the **only** publish path for the
internal topic from discovery. PollLoop SHALL NOT bypass it.

#### Scenario: PollLoop inserts new files as DISCOVERED

- **GIVEN** a file at the transport source not yet in `discovery_files`
- **WHEN** PollLoop's next tick processes the location
- **THEN** a row is inserted with `status = DISCOVERED`,
  `bundle_id = uuid`, `discovered_at = now()`
- **AND** PollLoop does not publish to Kafka

#### Scenario: PollLoop is idempotent on re-seeing a known file

- **GIVEN** a file already represented in `discovery_files` by a row
  matching `(location_id, filename, mtime)`
- **WHEN** PollLoop re-encounters it
- **THEN** the INSERT is a conflict no-op
- **AND** no new row is created
- **AND** no error is raised

#### Scenario: NotificationPublisher publishes in discovered_at order

- **GIVEN** three `DISCOVERED` rows for one location with
  `discovered_at` values A < B < C
- **WHEN** NotificationPublisher's tick processes the location
- **THEN** events are published in order A, B, C
- **AND** each event's broker ack is awaited before the next publish
- **AND** rows transition to `NOTIFIED` in the same order

#### Scenario: NotificationPublisher respects per-tick batch cap

- **GIVEN** N `DISCOVERED` rows for one location where N > batch cap
- **WHEN** NotificationPublisher's tick processes the location
- **THEN** at most `batch cap` rows are published this tick
- **AND** the remainder are processed on subsequent ticks in
  `discovered_at` order

#### Scenario: NotificationPublisher leaves row DISCOVERED on publish failure

- **GIVEN** a `DISCOVERED` row whose publish to Kafka fails (broker
  unavailable, network error, etc.)
- **WHEN** NotificationPublisher handles the failure
- **THEN** the row remains in `DISCOVERED` (no transition)
- **AND** `notified_at` remains NULL
- **AND** the next tick retries the publish

#### Scenario: PollLoop crash between INSERT and next tick is handled by NotificationPublisher

- **GIVEN** PollLoop has INSERTed `DISCOVERED` rows and the process
  crashes before the NotificationPublisher tick runs
- **WHEN** the runtime restarts
- **THEN** NotificationPublisher picks up the `DISCOVERED` rows from the
  table
- **AND** publishes them in `discovered_at` order
- **AND** transitions them to `NOTIFIED`

#### Scenario: PollLoop never publishes directly

- **GIVEN** a fresh INSERT by PollLoop
- **WHEN** the INSERT succeeds
- **THEN** no `FileDiscoveredEvent` is published by PollLoop
- **AND** publication is deferred to NotificationPublisher's next tick

### Requirement: discovery_files schema and ownership

The discovery subsystem SHALL own a `discovery_files` table with the
following schema:

```
discovery_files
  bundle_id       UUID PRIMARY KEY
  location_id     text NOT NULL
  filename        text NOT NULL
  mtime           timestamptz NOT NULL
  size            bigint NOT NULL
  status          text NOT NULL CHECK (status IN ('DISCOVERED', 'NOTIFIED'))
  discovered_at   timestamptz NOT NULL DEFAULT now()
  notified_at     timestamptz NULL
  UNIQUE (location_id, filename, mtime)
  INDEX idx_discovery_files_pending (location_id, status, discovered_at)
    WHERE status = 'DISCOVERED'
```

The migration SHALL ship with the library at `discovery/db/migrations/`
on the classpath. Discovery's runtime initialisation SHALL apply this
migration against the configured `db.discovery` connection.

#### Scenario: Migration creates discovery_files

- **GIVEN** a fresh database configured as `db.discovery`
- **WHEN** discovery's runtime initialisation runs
- **THEN** `discovery_files` exists with the documented schema
- **AND** the `idx_discovery_files_pending` index is present
- **AND** the CHECK constraint enforces the allowed status values

#### Scenario: Migration is idempotent

- **GIVEN** a database where `discovery_files` already exists with the
  current schema
- **WHEN** discovery's runtime initialisation runs again
- **THEN** the migration is a no-op
- **AND** initialisation completes successfully

#### Scenario: Pipeline does not query discovery_files

- **GIVEN** a running pipeline subsystem
- **WHEN** pipeline's queries are inspected
- **THEN** no query references the `discovery_files` table

### Requirement: ftp_file_state is owned by pipeline and keyed on bundleId

The pipeline subsystem SHALL own `ftp_file_state`. The schema SHALL be
keyed on `bundle_id`:

```
ftp_file_state
  bundle_id     UUID PRIMARY KEY
  location_id   text NOT NULL
  filename      text NOT NULL
  mtime         timestamptz NOT NULL
  size          bigint NOT NULL
  status        text NOT NULL CHECK (status IN
                  ('DOWNLOADING', 'DOWNLOADED', 'PUBLISHED',
                   'FAILED', 'SKIPPED'))
  bucket_path   text NULL
  sha256        text NULL
  attempts      int NOT NULL DEFAULT 0
  last_error    text NULL
  created_at    timestamptz NOT NULL DEFAULT now()
  updated_at    timestamptz NOT NULL DEFAULT now()
  INDEX idx_ftp_file_state_location_status (location_id, status)
```

The schema SHALL NOT include a `DISCOVERED` status — pipeline's state
machine starts at `DOWNLOADING` (state machine introduced in change 5).
Inserts SHALL use `ON CONFLICT (bundle_id) DO NOTHING` to make duplicate
event consumption idempotent.

The migration SHALL ship at `pipeline/db/migrations/` on the classpath.
Pipeline's runtime initialisation SHALL apply this migration against the
configured `db.pipeline` connection.

#### Scenario: Migration creates ftp_file_state keyed on bundle_id

- **GIVEN** a fresh database configured as `db.pipeline`
- **WHEN** pipeline's runtime initialisation runs
- **THEN** `ftp_file_state` exists with `bundle_id` as primary key
- **AND** no `DISCOVERED` status is permitted by the CHECK constraint

#### Scenario: Duplicate event consumption is a no-op at pipeline

- **GIVEN** a `FileDiscoveredEvent` whose `bundleId` already exists in
  `ftp_file_state` (any status)
- **WHEN** pipeline consumes the event
- **THEN** the INSERT is a conflict no-op
- **AND** the existing row is unchanged
- **AND** no error is raised

#### Scenario: Discovery does not query ftp_file_state

- **GIVEN** a running discovery subsystem
- **WHEN** discovery's queries are inspected
- **THEN** no query references the `ftp_file_state` table

### Requirement: FileDiscoveredEvent has a stable public schema

`FileDiscoveredEvent` SHALL be defined as a public type in the `shared/`
package with the following fields:

```kotlin
data class FileDiscoveredEvent(
    val schemaVersion: String = "1.0",
    val bundleId: UUID,
    val locationId: String,
    val filename: String,
    val mtime: Instant,
    val size: Long,
    val discoveredAt: Instant,
)
```

The event SHALL be part of the library's documented public API surface
(codified in change 3).

#### Scenario: Event carries bundleId matching the discovery_files row

- **GIVEN** a `discovery_files` row with `bundle_id = X` published by
  NotificationPublisher
- **WHEN** the resulting `FileDiscoveredEvent` is inspected
- **THEN** its `bundleId` field equals `X`
- **AND** its `discoveredAt` equals the row's `discovered_at`

#### Scenario: Event schema version is 1.0 for this change

- **GIVEN** any `FileDiscoveredEvent` produced by this library version
- **WHEN** its `schemaVersion` is inspected
- **THEN** it equals `"1.0"`

### Requirement: Internal topic semantics

The library SHALL publish `FileDiscoveredEvent` to a single internal
Kafka topic with `partitionKey = locationId`. The topic name SHALL come
from configuration (`ftp.internalTopicName`); default
`lib-ftp-watcher.discovered`.

Per-location ordering of events on the topic SHALL be preserved by the
combination of partition key and NotificationPublisher's serial
publish-and-ack discipline within a location.

#### Scenario: Two events for the same location land on the same partition

- **GIVEN** two `FileDiscoveredEvent` instances with the same
  `locationId`
- **WHEN** both are published
- **THEN** both land on the same partition of the configured internal
  topic

#### Scenario: Topic name is configurable

- **GIVEN** a configuration with
  `ftp.internalTopicName = "custom-name"`
- **WHEN** NotificationPublisher publishes
- **THEN** events go to a Kafka topic named `custom-name`

### Requirement: Each factory validates only the slice of config it reads

The library SHALL implement three validation profiles, one per factory
(factories themselves are introduced in change 3; the validation rules
they apply are defined here):

- **Discovery profile** — validates `ftp.transport`,
  `ftp.internalTopicName`, `ftp.db.discovery`, and the discovery fields
  of each location (`id`, `source`, `pollIntervalSeconds`,
  `filenamePattern`). Tolerates absence of pipeline-only fields
  (`destination`, `topic`, `onFailure`, `backoff`, `maxAttempts`) and
  `ftp.db.pipeline`.
- **Pipeline profile** — validates `ftp.internalTopicName`,
  `ftp.db.pipeline`, and the pipeline fields of each location (`id`,
  `destination`, `topic`, `onFailure`, `backoff`, `maxAttempts`).
  Tolerates absence of `ftp.transport` and discovery-only fields.
- **Composite (Watcher) profile** — validates every field. Used when
  the composite factory is called.

Validation errors SHALL aggregate. A single validation pass reports all
violations for the active profile.

#### Scenario: Discovery profile accepts config missing pipeline fields

- **GIVEN** a HOCON config with `ftp.transport`, `ftp.internalTopicName`,
  `ftp.db.discovery`, and locations carrying only discovery fields (no
  `destination`, no `topic`, no `onFailure`)
- **WHEN** validated under the discovery profile
- **THEN** validation passes

#### Scenario: Pipeline profile accepts config missing discovery fields

- **GIVEN** a HOCON config with `ftp.internalTopicName`, `ftp.db.pipeline`,
  and locations carrying only pipeline fields (no `source`, no
  `pollIntervalSeconds`)
- **WHEN** validated under the pipeline profile
- **THEN** validation passes

#### Scenario: Composite profile rejects config missing any required field

- **GIVEN** a HOCON config missing `ftp.db.pipeline`
- **WHEN** validated under the composite profile
- **THEN** validation fails
- **AND** the error names `ftp.db.pipeline` as missing

#### Scenario: Discovery profile rejects missing transport block

- **GIVEN** a HOCON config without `ftp.transport`
- **WHEN** validated under the discovery profile
- **THEN** validation fails
- **AND** the error names `ftp.transport` as missing

### Requirement: testFixtures provides three LocalRunner factory functions

The library's `testFixtures` source set SHALL expose three factory
functions on `LocalRunner`:

- `LocalRunner.discoveryOnly(configPath)` — boots only the discovery
  subsystem.
- `LocalRunner.pipelineOnly(configPath)` — boots only the pipeline
  subsystem.
- `LocalRunner.both(configPath)` — boots both subsystems in one process.

testFixtures SHALL also provide reusable test doubles:
`StubSecretGetter` (Map-backed `SecretGetter`) and
`InMemoryDomainEventRouter` (in-process router substitute).

#### Scenario: LocalRunner.discoveryOnly starts only discovery

- **GIVEN** `LocalRunner.discoveryOnly(configPath)` invoked with a
  config valid under the discovery profile
- **WHEN** `start()` is called
- **THEN** discovery's PollLoop and NotificationPublisher are running
- **AND** no pipeline coroutines are running

#### Scenario: LocalRunner.pipelineOnly starts only pipeline

- **GIVEN** `LocalRunner.pipelineOnly(configPath)` invoked with a
  config valid under the pipeline profile
- **WHEN** `start()` is called
- **THEN** pipeline's event consumer is running (event-consumer
  implementation lands in change 5; in this change a stub binding
  suffices)
- **AND** no discovery coroutines are running

#### Scenario: LocalRunner.both starts both subsystems

- **GIVEN** `LocalRunner.both(configPath)` invoked with a config valid
  under the composite profile
- **WHEN** `start()` is called
- **THEN** discovery's PollLoop and NotificationPublisher are running
- **AND** pipeline's event consumer is running
- **AND** both subsystems share the in-process Kafka client / DB pool
  configuration declared in HOCON

#### Scenario: InMemoryDomainEventRouter is reusable in tests

- **GIVEN** a test that overrides the `DomainEventRouter` with
  `InMemoryDomainEventRouter`
- **WHEN** events are published to the in-memory router
- **THEN** they are observable to subscribers in-process
- **AND** no Kafka broker is required

## MODIFIED Requirements

### Requirement: Migration packaging

> Reframed from change 1: migrations are now per-subsystem and applied at
> each subsystem's runtime initialisation, against that subsystem's
> configured DB.

The library SHALL ship discovery's and pipeline's migrations separately,
each in its subsystem's resource path:

- `discovery/db/migrations/` — owned by discovery, applied against
  `ftp.db.discovery`.
- `pipeline/db/migrations/` — owned by pipeline, applied against
  `ftp.db.pipeline`.

Discovery-only deployments SHALL NOT apply pipeline's migrations and
vice versa. Composite deployments apply both, but each against its own
configured DB connection (which may or may not be the same physical
database).

#### Scenario: Discovery-only deployment applies only discovery migrations

- **GIVEN** a deployment running only `LocalRunner.discoveryOnly`
- **WHEN** the runtime initialises
- **THEN** `discovery_files` is created in `ftp.db.discovery`
- **AND** `ftp_file_state` is NOT created in either database

#### Scenario: Composite deployment with single physical DB applies both migrations

- **GIVEN** a HOCON config where `ftp.db.discovery = ${ftp.db.pipeline}`
- **AND** `LocalRunner.both` is invoked
- **WHEN** the runtime initialises
- **THEN** `discovery_files` is created in the shared database
- **AND** `ftp_file_state` is created in the same shared database
- **AND** neither migration interferes with the other

#### Scenario: Composite deployment with two physical DBs applies migrations independently

- **GIVEN** a HOCON config where `ftp.db.discovery` and `ftp.db.pipeline`
  point at different physical databases
- **AND** `LocalRunner.both` is invoked
- **WHEN** the runtime initialises
- **THEN** `discovery_files` is created only in the discovery DB
- **AND** `ftp_file_state` is created only in the pipeline DB

### Requirement: Operational status is exposed as routable HTTP routes

> Reframed from change 1 (and from earlier `restructure-as-library`
> drafts): status routes split per subsystem. Each subsystem exposes its
> own routable module. The library does not mount, host, or expose the
> routes on its own; mounting is the consumer's responsibility.

The pipeline subsystem SHALL expose a routable status module covering
per-location worker state, counts, and last-observed timestamps. In this
change, the payload is a stub; change 5 populates it.

The discovery subsystem SHALL expose a routable status module covering:

- Configured locations
- Per-location: last `PollLoop` tick timestamp, last successful
  NotificationPublisher publish timestamp, oldest unpublished
  `discovered_at`, count of `DISCOVERED` rows (publisher backlog)

Consumers in composite topology mount both modules at distinct paths
they choose.

#### Scenario: Discovery status reports configured locations and zero backlog at start

- **GIVEN** a fresh discovery-only deployment with `N` configured locations
- **WHEN** the discovery status module is queried
- **THEN** the response includes `"locations_configured": N`
- **AND** each location's entry shows `backlog: 0` and null
  last-poll/last-publish timestamps

#### Scenario: Discovery status reports backlog when DISCOVERED rows exist

- **GIVEN** a discovery deployment with M rows in `DISCOVERED` for
  location L
- **WHEN** the discovery status module is queried
- **THEN** location L's entry shows `backlog: M`
- **AND** `oldest_unpublished_discovered_at` reflects the oldest such row

#### Scenario: Pipeline status module is mountable independently

- **GIVEN** a pipeline-only deployment
- **WHEN** the pipeline status module is mounted by the consumer
- **THEN** it is queryable at the consumer's chosen path
- **AND** the discovery status module is not present
