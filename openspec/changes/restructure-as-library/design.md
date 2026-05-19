# Design — Restructure as Library

## House patterns applied

Existing platform-library dependencies are unchanged: `service-bootstrap`,
`application-config`, `secret-getter`, `db-toolkit`, `observability`.

This change **catalogues `DomainEventRouter`** in `project.md` per the
on-demand library discovery rule. `DomainEventRouter` will be used by:

- Discovery's NotificationPublisher to publish `FileDiscoveredEvent` to
  the internal topic (used in change 5).
- Pipeline's event consumer to subscribe to the internal topic (used in
  change 5).
- Pipeline's external publisher to publish `FileReceivedEvent` to each
  location's external topic (used in change 5).

The catalogue lives here so the contract starts here. Implementations
land in change 5.

Confirm at implementation start:

- `db-toolkit` supports library-packaged classpath migrations from a
  sub-path (`discovery/db/migrations/`, `pipeline/db/migrations/`).
- `service-bootstrap` can be consumed by a library that contributes a
  readiness check without itself being a deployable service.
- The platform's routable-module API for HTTP routes.
- `DomainEventRouter` coordinates, API, and shutdown lifecycle.
- `testFixtures` artifacts can be published to the org's Artifactory.

## Both topologies side-by-side

```
                  Composite topology                     Split topology
                  ──────────────────                     ──────────────

       ┌──────────────────────────────┐         ┌────────────────────────┐
       │   WatcherRuntime (one proc)  │         │  DiscoveryRuntime      │
       │                              │         │   ┌──────────────┐     │
       │  ┌────────────────────────┐  │         │   │  PollLoop    │     │
       │  │  DiscoveryRuntime      │  │         │   │  per-loc     │     │
       │  │   PollLoop             │  │         │   └──────┬───────┘     │
       │  │   NotificationPublisher│  │         │   ┌──────▼───────┐     │
       │  │     ──▶ discovery_files│  │         │   │ discovery_   │     │
       │  └────────────┬───────────┘  │         │   │   files      │     │
       │               │              │         │   └──────────────┘     │
       │               │              │         │   ┌──────────────┐     │
       │               ▼              │         │   │NotifPublisher│     │
       │     ┌──────────────────┐     │         │   └──────┬───────┘     │
       │     │ Kafka:           │     │         └──────────┼─────────────┘
       │     │ internalTopic    │     │                    │
       │     │ partition by     │     │                    ▼
       │     │ locationId       │     │         ┌────────────────────────┐
       │     └────────┬─────────┘     │         │ Kafka: internal topic  │
       │              │               │         │ partition by locationId│
       │              ▼               │         └──────────┬─────────────┘
       │  ┌────────────────────────┐  │                    │
       │  │  PipelineRuntime       │  │                    ▼
       │  │   Event consumer       │  │         ┌────────────────────────┐
       │  │   Delivery worker      │  │         │  PipelineRuntime       │
       │  │     ──▶ ftp_file_state │  │         │   Event consumer       │
       │  │     ──▶ GCS upload     │  │         │   Delivery worker      │
       │  │     ──▶ external topic │  │         │     ──▶ ftp_file_state │
       │  └────────────────────────┘  │         │     ──▶ GCS upload     │
       └──────────────────────────────┘         │     ──▶ external topic │
                                                └────────────────────────┘
```

Same code, different processes. The library doesn't know or care which
topology it's running in.

## Package structure

```
lib-ftp-location-watcher/
├── build.gradle.kts                       # java-test-fixtures enabled
├── src/
│   ├── main/
│   │   ├── kotlin/com/.../ftpwatcher/
│   │   │   ├── shared/                    # types used by both subsystems
│   │   │   │   ├── config/                # WatcherConfig + nested types
│   │   │   │   ├── events/                # FileDiscoveredEvent
│   │   │   │   ├── transport/             # FileTransport interface (lands in change 4)
│   │   │   │   └── validation/            # Per-profile validators
│   │   │   ├── discovery/                 # NOT importable from pipeline
│   │   │   │   ├── PollLoop.kt            # skeleton; impl in change 5
│   │   │   │   ├── NotificationPublisher.kt   # skeleton; impl in change 5
│   │   │   │   ├── DiscoveryStateStore.kt # discovery_files access
│   │   │   │   ├── DiscoveryStatusRoutes.kt   # routable module
│   │   │   │   └── internal/              # internal wiring
│   │   │   └── pipeline/                  # NOT importable from discovery
│   │   │       ├── EventConsumer.kt       # skeleton; impl in change 5
│   │   │       ├── DeliveryWorker.kt      # skeleton; impl in change 5
│   │   │       ├── PipelineStateStore.kt  # ftp_file_state access
│   │   │       ├── PipelineStatusRoutes.kt    # routable module
│   │   │       └── internal/
│   │   └── resources/
│   │       ├── discovery/db/migrations/
│   │       │   └── V001__create_discovery_files.sql
│   │       └── pipeline/db/migrations/
│   │           └── V001__create_ftp_file_state.sql
│   ├── test/                              # library's own tests
│   └── testFixtures/
│       ├── kotlin/com/.../ftpwatcher/testfixtures/
│       │   ├── LocalRunner.kt             # three factory functions
│       │   ├── LocalRunnerMain.kt         # main() for :runLocal
│       │   └── doubles/
│       │       ├── StubSecretGetter.kt
│       │       └── InMemoryDomainEventRouter.kt
│       └── resources/
│           └── localrunner.example.conf   # annotated HOCON
```

The import boundary between `discovery/` and `pipeline/` is enforced by
a build-time check (Detekt rule or a simple test scanning import
statements). This is the structural guarantee that justifies "separately
bootstrappable."

## State table schemas

### `discovery_files`

```sql
CREATE TABLE discovery_files (
    bundle_id      UUID PRIMARY KEY,
    location_id    TEXT NOT NULL,
    filename       TEXT NOT NULL,
    mtime          TIMESTAMPTZ NOT NULL,
    size           BIGINT NOT NULL,
    status         TEXT NOT NULL CHECK (status IN ('DISCOVERED', 'NOTIFIED')),
    discovered_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    notified_at    TIMESTAMPTZ NULL,
    UNIQUE (location_id, filename, mtime)
);

CREATE INDEX idx_discovery_files_pending
    ON discovery_files (location_id, discovered_at)
    WHERE status = 'DISCOVERED';
```

Notes:

- `bundle_id` is the primary key. Minted by PollLoop on INSERT.
  Carried through into `FileDiscoveredEvent` and (eventually) into
  pipeline's `ftp_file_state`.
- The `UNIQUE (location_id, filename, mtime)` constraint is the
  natural-key dedup for PollLoop's `ON CONFLICT DO NOTHING`.
- The partial index on `status = 'DISCOVERED'` keeps the
  NotificationPublisher query fast even when the table has accumulated
  millions of `NOTIFIED` rows.
- `notified_at` is observability sugar (latest publish time, "how stale
  is the publish queue"). Not load-bearing for correctness — the status
  column tells you everything.

### `ftp_file_state`

```sql
CREATE TABLE ftp_file_state (
    bundle_id     UUID PRIMARY KEY,
    location_id   TEXT NOT NULL,
    filename      TEXT NOT NULL,
    mtime         TIMESTAMPTZ NOT NULL,
    size          BIGINT NOT NULL,
    status        TEXT NOT NULL CHECK (status IN
                    ('DOWNLOADING', 'DOWNLOADED', 'PUBLISHED',
                     'FAILED', 'SKIPPED')),
    bucket_path   TEXT NULL,
    sha256        TEXT NULL,
    attempts      INT NOT NULL DEFAULT 0,
    last_error    TEXT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ftp_file_state_location_status
    ON ftp_file_state (location_id, status);
```

Notes:

- `bundle_id` is the primary key — same UUID minted by discovery's
  PollLoop, carried through the event. Pipeline never generates a
  bundle_id.
- No `DISCOVERED` state. Pipeline's state machine (defined in change 5)
  starts at `DOWNLOADING`. Pipeline accepts an event, inserts the row at
  `DOWNLOADING`, and proceeds.
- `ON CONFLICT (bundle_id) DO NOTHING` on insert makes duplicate event
  consumption a safe no-op.
- The status column dropped the change-1 `DISCOVERED` value — verify
  during the regression pass that no other code references it.

## `FileDiscoveredEvent` schema

```kotlin
package com.example.ftpwatcher.shared.events

import java.time.Instant
import java.util.UUID

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

Field rationale:

- `schemaVersion` — explicit so consumers can dispatch on version when
  future event-shape changes arrive.
- `bundleId` — the durable identifier and pipeline's primary key. The
  whole reason the event exists is to tell pipeline about this ID.
- `locationId` — partition key on the topic; also used by pipeline for
  per-location config lookup.
- `filename`, `mtime`, `size` — enough for pipeline to make logging /
  metrics decisions without a DB roundtrip.
- `discoveredAt` — preserves the original discovery timestamp through
  Kafka. Useful for end-to-end latency metrics.

The event does not include `transport` details or credentials.
Pipeline looks those up from config by `locationId`.

## Why two cooperating loops in discovery (not one)

We considered three approaches:

1. **PollLoop publishes inline.** PollLoop INSERTs then publishes in the
   same pass. Simplest model on paper. Problems: every PollLoop tick
   either succeeds completely or leaks partial state (INSERT but no
   publish). Retry logic creeps into PollLoop. Ordering correctness
   requires `EXISTS`-checks against the backlog before each inline
   publish.

2. **PollLoop publishes inline; "catch-up" handles only failures.**
   The model we initially drafted. PollLoop publishes when there's no
   backlog; defers to catch-up when there is. Correct, but two distinct
   publish paths means two places to maintain ordering invariants, and
   the `EXISTS` check per insert is a real cost on busy locations.

3. **PollLoop never publishes; NotificationPublisher is the sole publish
   path.** What we landed on. PollLoop is a pure data writer.
   NotificationPublisher is the sole producer. Ordering correctness is
   structural — there's exactly one publish path, ordered by
   `discovered_at`, serialized per location. Failure handling is
   uniform: a failed publish leaves the row `DISCOVERED`; next tick
   retries.

The trade-off: latency. With a 60-second NotificationPublisher tick, a
file inserted at second 0 publishes around second 60 in the worst case.
For FTP-polled feeds where poll intervals are typically 60s–5min, this
latency is invisible. If sub-second event latency becomes a requirement,
a future change can add in-process signaling between PollLoop and
NotificationPublisher to wake the publisher immediately on new INSERTs.
The contract — "NotificationPublisher is the sole publish path" — is
preserved; only the wait-loop changes.

### NotificationPublisher tick interval and batch cap

Defaults:

- Tick interval: 60 seconds.
- Per-tick batch cap per location: 100 rows.

These are configurable. The combination caps Kafka publish rate at
~100 events per location per minute per process, which is far above any
reasonable file-arrival rate for FTP-distributed reference data.

### Why per-location loops (not one global loop)

Per-location coroutines for both PollLoop and NotificationPublisher
match the "locations are independent" theme that runs through the design.
The cost is N coroutines instead of 1 — trivial. The benefit is that
one slow location (an FTP server taking 30 seconds to list, or a stuck
Kafka publish) doesn't block other locations.

## Validation strictness per factory

The validation rule is "each factory validates only what it reads."
Implementation:

```kotlin
internal object ConfigValidator {

    fun validateForDiscovery(config: WatcherConfig): ValidationResult {
        val errors = mutableListOf<ValidationError>()
        validateTransport(config.transport, errors)
        validateInternalTopicName(config.internalTopicName, errors)
        validateDiscoveryDb(config.db.discovery, errors)
        config.locations.forEachIndexed { i, loc ->
            validateLocationDiscoveryFields(loc, i, errors)
        }
        return ValidationResult(errors)
    }

    fun validateForPipeline(config: WatcherConfig): ValidationResult { ... }
    fun validateForComposite(config: WatcherConfig): ValidationResult { ... }
}
```

Each factory (defined in change 3) calls the matching validator. Errors
aggregate. Single-pass validation per factory call.

The location entry's fields are all *optional at the type level* in
`WatcherConfig`. Validators enforce required-ness per profile. This
keeps the config type uniform across topologies and pushes the
"required vs. optional" question to the place that can answer it (the
factory in use).

## Two pools in composite — rationale

`WatcherRuntime` (composite, defined in change 3) opens two separate
`db-toolkit` connection pools — one from `db.discovery`, one from
`db.pipeline` — even when both blocks point at the same physical
database. Three reasons:

1. **Subsystem independence.** Each subsystem builds and owns its own
   pool from its own config block. No special-casing for "are these the
   same?"
2. **Future-proof against split DBs.** If a consuming service ever
   points the two subsystems at different databases, no code change is
   needed.
3. **Independent tuning.** Discovery's DB load profile (frequent small
   inserts, occasional reads) differs from pipeline's (frequent state
   transitions on a smaller hot set). Pool sizing can differ if needed.

The cost is N+M connections per process instead of max(N, M). Small,
benign. Consumers who genuinely care about composite pool count can
build a custom assembly via the factory builder in change 3.

## testFixtures shape

`LocalRunner.kt`:

```kotlin
package com.example.ftpwatcher.testfixtures

class LocalRunner private constructor(...) {

    fun start(): AutoCloseable { ... }

    companion object {
        fun discoveryOnly(configPath: String): LocalRunner { ... }
        fun pipelineOnly(configPath: String): LocalRunner { ... }
        fun both(configPath: String): LocalRunner { ... }
    }
}
```

`LocalRunnerMain.kt`:

```kotlin
fun main(args: Array<String>) {
    val mode = args.getOrNull(0) ?: "both"
    val configPath = args.getOrNull(1) ?: "localrunner.example.conf"
    val runner = when (mode) {
        "discovery" -> LocalRunner.discoveryOnly(configPath)
        "pipeline" -> LocalRunner.pipelineOnly(configPath)
        else -> LocalRunner.both(configPath)
    }
    val closeable = runner.start()
    Runtime.getRuntime().addShutdownHook(Thread { closeable.close() })
    // join indefinitely
}
```

Gradle task `:runLocal` invokes `LocalRunnerMain`.

`localrunner.example.conf`:

```hocon
ftp {
  internalTopicName = "lib-ftp-watcher.discovered"
  transport {
    type = local
    # ...
  }
  locations = [
    {
      id = "stuttgart-warrants"
      source { type = local, watchDir = "/tmp/watch/stuttgart" }
      pollIntervalSeconds = 10
      filenamePattern = "*.zip"
      # pipeline-side fields:
      destination { bucket = "dev-bucket", pathPrefix = "stuttgart/" }
      topic = "dev.posttrade.stuttgart.received"
      onFailure = HALT
      backoff { initialSeconds = 5, multiplier = 2.0, maxSeconds = 300 }
      maxAttempts = 5
    }
  ]
  db {
    # HOCON substitution: one physical DB for local dev
    common = {
      host = "localhost"
      port = 5432
      database = "watcher_dev"
      username = "watcher"
      passwordSecretKey = "watcher.db.password"
      pool { maxSize = 10 }
    }
    discovery = ${ftp.db.common}
    pipeline = ${ftp.db.common}
  }
}
```

`InMemoryDomainEventRouter`:

```kotlin
class InMemoryDomainEventRouter : DomainEventRouter {

    private val subscribers = mutableMapOf<String, MutableList<(Any) -> Unit>>()

    override fun publish(topic: String, partitionKey: String, payload: Any) {
        subscribers[topic].orEmpty().forEach { it(payload) }
    }

    override fun subscribe(topic: String, handler: (Any) -> Unit) {
        subscribers.getOrPut(topic) { mutableListOf() }.add(handler)
    }
}
```

The exact `DomainEventRouter` API is confirmed at implementation start;
the in-memory shape mirrors it.

## Regression strategy

Every scenario from change 1 (`add-watcher-foundation`) SHALL still pass
under this restructure. The migration of change-1 tests into the
library module is the pre-merge gate. Any failure halts the change.

Specifically:

- Config schema validation rules from change 1 still apply (now via the
  composite validator profile when `WatcherFactory.create` is the entry).
- Readiness check still transitions to `READY` after migrations apply.
- The Docker Compose dev stack still brings up Postgres, fake-gcs,
  Redpanda; `LocalRunner.both` reaches `READY`.

## Risks and open questions

- **`db-toolkit` library-packaged migrations from a sub-path.**
  Migrations under `discovery/db/migrations/` and
  `pipeline/db/migrations/` require `db-toolkit` to accept a
  configurable classpath path. If it expects a single fixed path, the
  library extracts resources to a temp directory or has each subsystem
  use its own migration runner instance.
- **`service-bootstrap` library-consumption.** Confirm the library can
  consume bootstrap features (readiness check registration) without
  itself being a deployable service.
- **`DomainEventRouter` API specifics.** The in-memory router substitute
  must mirror the real API. Confirm at implementation start.
- **testFixtures publishing to Artifactory.** Confirm supported.
- **Build-time import boundary check.** Detekt rule vs. a simple
  test-scanning approach. Decide at implementation; both work, Detekt is
  more discoverable.
- **NotificationPublisher tick interval default.** 60 seconds is
  proposed. If FTP poll intervals are routinely much shorter (e.g.
  10 seconds for testing), a longer publisher tick creates noticeable
  lag in local dev. The example HOCON can override to a shorter tick
  for `LocalRunner`.
