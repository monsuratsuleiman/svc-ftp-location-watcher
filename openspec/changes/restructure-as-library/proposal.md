# Restructure as Library

## Why

The watcher (scaffolded as a single service in `add-watcher-foundation`)
needs to become a Gradle library consumed by multiple functional-domain
services (e.g. `svc-posttrade-ftp-location-watcher`,
`svc-marketdata-ftp-location-watcher`). Two concerns land together in this
change:

1. **Mechanical restructure.** svc → lib module rename,
   `java-test-fixtures` plugin enabled, production `main()` removed,
   library-packaged migrations applied at runtime initialisation.
2. **Architectural seam.** The library exposes discovery and pipeline as
   separately bootstrappable subsystems. Consuming services choose their
   topology — composite (one process) or split (two services) — by which
   factory they call from `main()`. Not by config. The seam is an
   internal Kafka topic.

The architectural seam belongs *here*, before the public API solidifies
in the next change. Adding it later would be a breaking API change;
adding it now is a free design decision.

The discovery and pipeline implementations land in change 5
(`add-pipeline`). This change establishes the package layout, state-table
schemas, the internal event schema, the validation rules, and the
testFixtures contract that change 5 fills in.

## The model

Discovery and pipeline are independent subsystems coupled only by an
internal Kafka topic carrying `FileDiscoveredEvent`.

### Discovery has two cooperating loops per location

- **PollLoop** polls the transport, calls `transport.list()`, and inserts
  newly-seen files into `discovery_files` with `status = DISCOVERED`.
  Idempotent via `ON CONFLICT (location_id, filename, mtime) DO NOTHING`.
  **PollLoop never publishes to Kafka.**
- **NotificationPublisher** reads `discovery_files` for that location
  ordered by `discovered_at ASC`, publishes `FileDiscoveredEvent` for each
  `DISCOVERED` row, waits for the broker ack, then transitions the row to
  `NOTIFIED` with `notified_at = now()`. Serial publish-and-ack per row.
  Per-tick batch cap (default 100) prevents one location's backlog from
  monopolising a tick. Publish failures leave the row `DISCOVERED`; the
  next tick retries.

Splitting these responsibilities makes ordering correctness structural:
there is exactly one publish path per location, so messages reach the
internal topic in `discovered_at` order with no race. PollLoop is a pure
data writer; NotificationPublisher is the sole producer.

### Pipeline is event-driven, idempotent on `bundleId`

Pipeline consumes `FileDiscoveredEvent`, looks up the corresponding row,
and progresses it through the state machine introduced in change 5
(`DOWNLOADING → DOWNLOADED → PUBLISHED`, with `FAILED` and `SKIPPED`
terminal off-ramps). Pipeline's state table (`ftp_file_state`) is keyed
on `bundleId` and uses `ON CONFLICT (bundle_id) DO NOTHING` to swallow
duplicate event consumption without state corruption.

Pipeline has no `DISCOVERED` state; rows enter the table when the
pipeline accepts an event, not when discovery sees a file. The two
subsystems each own their own slice of the lifecycle.

### Per-subsystem state tables; per-subsystem migrations

Discovery owns `discovery_files`. Pipeline owns `ftp_file_state`. Each
subsystem ships its own migration files and applies them against its
own configured DB at runtime initialisation.

For Day 1, most consumers will point both subsystems at the same physical
database via HOCON substitution. The schemas don't collide. Pointing
them at separate databases is supported by the library — it's a config
choice, not a code change. The library opens two separate connection
pools regardless (one per subsystem, from each subsystem's own config
block).

### Internal topic

Single Kafka topic, `partitionKey = locationId`, default name
`lib-ftp-watcher.discovered`, configurable. The same `DomainEventRouter`
platform library publishes both internal (`FileDiscoveredEvent`) and
external (`FileReceivedEvent` from change 5). `DomainEventRouter` is
catalogued in `project.md` as part of this change per the on-demand
library discovery rule.

### `FileDiscoveredEvent` is a public type

Defined in the library's `shared/` package. Carries the `bundleId`
(minted by PollLoop on INSERT), `locationId`, `filename`, `mtime`,
`size`, `discoveredAt`. Stable schema (`schemaVersion = "1.0"` for this
change).

## Config schema

Single uniform block. Mode is determined by factory choice, not config:

```hocon
ftp {
  internalTopicName = "lib-ftp-watcher.discovered"
  transport { ... }                              # discovery uses
  locations = [
    {
      id, source, pollIntervalSeconds, filenamePattern,    # discovery
      destination, topic, onFailure, backoff, maxAttempts  # pipeline
    }
  ]
  db {
    discovery { ... }     # full DB block; identical shape to pipeline
    pipeline  { ... }     # full DB block; separately overridable
  }
}
```

`db.discovery` and `db.pipeline` have identical schemas. Day-1 single-DB
deployments are a one-line HOCON substitution
(`ftp.db.discovery = ${ftp.db.pipeline}`). The library never assumes they
point at the same place.

Each factory validates only the slice it reads:

- `DiscoveryFactory.create(config)` — validates `transport`,
  `internalTopicName`, `db.discovery`, and the discovery fields of each
  location. Tolerates absence of pipeline-only fields.
- `PipelineFactory.create(config)` — validates `internalTopicName`,
  `db.pipeline`, and the pipeline fields of each location. Tolerates
  absence of discovery-only fields and `transport`.
- `WatcherFactory.create(config)` — validates everything (composite needs
  the full picture).

The factories themselves land in change 3. This change specifies the
validation rules they implement.

Cross-HOCON consistency in split deployments (e.g. matching
`internalTopicName` across two files) is documented in README, not
enforced by library side-channel.

## What changes in this change

### Library mechanics

- Gradle module renamed svc → lib. Maven coordinates updated.
- `java-test-fixtures` plugin enabled.
- Production `main()` removed.
- All production types not in the documented public surface marked
  `internal`.

### Architecture

- Two production packages: `discovery/`, `pipeline/`, plus `shared/`.
  No imports between `discovery/` and `pipeline/` (only via `shared/`).
- `FileDiscoveredEvent` defined in `shared/`.
- `discovery_files` migration defined and packaged in `discovery/`.
- `ftp_file_state` migration moved from change 1's location into
  `pipeline/` resources. The schema gains `bundle_id UUID PRIMARY KEY`
  and drops the natural-key PK (becomes a non-unique index for queries).
- The status column allowed values are updated: `discovery_files` enum
  is `DISCOVERED | NOTIFIED`; `ftp_file_state` enum drops `DISCOVERED`
  (pipeline state machine starts at `DOWNLOADING`).
- Each subsystem's runtime initialisation applies its own migrations
  against its own configured DB.

### Status routes

- Pipeline owns the rich status routes (per-location worker state,
  counts, timestamps) as a routable module. Currently a stub; populated
  in change 5.
- Discovery owns a minimal status routes module: configured locations,
  last poll timestamp per location, oldest unpublished `discovered_at`
  per location, publisher backlog count.
- Consumers mount each module at distinct paths in composite topology.

### testFixtures

- `LocalRunner` gets three factory functions:
  - `LocalRunner.discoveryOnly(configPath)`
  - `LocalRunner.pipelineOnly(configPath)`
  - `LocalRunner.both(configPath)` — default for local dev
- Reusable test doubles: `StubSecretGetter`, `InMemoryDomainEventRouter`
  (in-process router substitute for unit tests).
- Annotated `localrunner.example.conf` covering all three modes.

## What doesn't change

- No discovery or pipeline implementations. The two packages are
  skeletons; change 5 fills them in. PollLoop and NotificationPublisher
  are *specified* here but *built* in change 5.
- No public factory API. Change 3 codifies `DiscoveryFactory`,
  `PipelineFactory`, `WatcherFactory`.
- No new platform-library dependencies beyond cataloguing
  `DomainEventRouter`.
- Validation rules from change 1 are preserved; this change adds new
  ones (per-factory validation strictness, `internalTopicName` presence).

## What this means for downstream changes

- **Change 3 (`add-library-public-api`)** exposes three factories:
  `DiscoveryFactory`, `PipelineFactory`, `WatcherFactory`. Each takes the
  unified `WatcherConfig`. Builder pattern available for each. Topic
  uniqueness validation extends to `internalTopicName` vs each location's
  external `topic`.
- **Change 5 (`add-pipeline`)** is renamed conceptually to "add discovery
  and pipeline." Internal split:
  - Discovery: PollLoop, NotificationPublisher, both per-location.
  - Pipeline: event consumer, delivery worker, state machine, GCS
    upload, external Kafka publish.
- **The single-instance assumption in change 5's design** now applies
  specifically to the **pipeline** subsystem. Discovery is naturally
  horizontally scalable: PollLoop INSERTs are dedup'd by the unique
  constraint, NotificationPublisher's `UPDATE` is dedup'd by the
  `DISCOVERED → NOTIFIED` transition. Multiple discovery replicas could
  poll the same location concurrently with no corruption (just wasted
  work). Pipeline retains the day-1 single-instance simplification.

## Impact

- Published artifact: `lib-ftp-location-watcher`. Not deployable on its
  own.
- Consuming services choose their topology by factory call. Same HOCON
  works for all three modes.
- Local development continues via `LocalRunner.both(...)` by default;
  the other two factories support subsystem-isolated testing.
- The library's public API surface is established as small and
  deliberate (everything except documented types is `internal`).

## Open questions resolved at implementation start

- Library Maven coordinates (`<TBD>` in `project.md`).
- Library versioning scheme.
- Confirmation that `db-toolkit` supports library-packaged classpath
  migrations from a sub-path.
- Confirmation that `service-bootstrap` can be consumed by a library.
- Platform's routable-module API for the status routes.
- `DomainEventRouter` Maven coordinates and API.
- NotificationPublisher tick interval default (proposed: 60s; configurable).
- NotificationPublisher per-tick batch cap default (proposed: 100;
  configurable).
- Whether `internalTopicName` needs a different default to avoid
  collision across multiple `lib-ftp-watcher` deployments sharing a
  Kafka cluster (proposed: keep the simple default; consumers override
  when needed).
