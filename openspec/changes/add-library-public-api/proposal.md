# Add Library Public API

## Why

After `restructure-as-library`, the watcher is a Gradle library with two
subsystems (`discovery/`, `pipeline/`) and a documented architectural
seam. But consuming services still can't construct a runtime without
reaching into `internal` wiring — the only construction path today is the
testFixtures `LocalRunner`, which is not a production API.

This change introduces the **public API contract**: the small, deliberate
set of types that consuming services depend on from their `main()`. Two
factories, two runtimes, two builders, plus a build-time guarantee that
nothing else is public by accident.

Equally important: this change locks in **what the public API is *not***.
Adding new public types in later changes requires an explicit update to
the documented surface, gated by a build-time assertion. Drift becomes
detectable, not invisible.

The factories themselves are minimal at this point — discovery and
pipeline don't yet *do* anything. PollLoop, NotificationPublisher, the
event consumer, the delivery worker all land in change 5. But the
construction surface is stable from this change forward.

## What changes

### Two factories, two runtimes, two builders

For each subsystem:

- A factory with `create(config)` for the production path and
  `builder(config)` for tests and special-case deployments.
- A runtime with `start()`, `stop()`, `isRunning()`.
- A builder that fluently accepts component overrides and produces
  a runtime via `build()`.

```kotlin
DiscoveryFactory.create(config) → DiscoveryRuntime
DiscoveryFactory.builder(config) → DiscoveryRuntimeBuilder

PipelineFactory.create(config) → PipelineRuntime
PipelineFactory.builder(config) → PipelineRuntimeBuilder
```

No `WatcherFactory`. The composite topology is just "call both
factories in your `main()`":

```kotlin
fun main() {
    val config = ApplicationConfig.load<WatcherConfig>()
    val discovery = DiscoveryFactory.create(config)
    val pipeline = PipelineFactory.create(config)

    Runtime.getRuntime().addShutdownHook(Thread {
        discovery.stop()
        pipeline.stop()
    })

    discovery.start()
    pipeline.start()
}
```

A wrapper factory would add no behaviour beyond these five lines.
Omitting it keeps the API smaller.

### Builder overrides

Both builders share the same three overrides — the components that have
meaningful test substitutes today:

- `withSecretGetter(SecretGetter)` — inject a test secret store
- `withClock(Clock)` — deterministic time in tests
- `withDomainEventRouter(DomainEventRouter)` — inject `InMemoryDomainEventRouter`
  to avoid Kafka in unit tests

Additional overrides land alongside the abstractions they swap:

- `withTransport(FileTransport)` on `DiscoveryRuntimeBuilder` — added in
  change 4 (`add-local-transport`) when `FileTransport` becomes a real
  type.
- `withDestination(FileDestination)` on `PipelineRuntimeBuilder` — added
  in the destination change (after change 4) when `FileDestination`
  becomes a real type.

Each new override is an explicit, documented expansion of the public
surface, gated by the surface-test update.

### Lifecycle semantics

`start()` is **non-blocking**. It performs the synchronous work that
must succeed for the runtime to be considered started (DB migration
apply, connection pool open, Kafka client / consumer creation,
subscription), then launches background coroutines and returns.

If any synchronous startup work fails, `start()` throws. The runtime is
left in a "never started" state — `isRunning()` returns false, `stop()`
is a no-op. Consumers can rebuild and retry, or escalate.

After successful `start()`, transient failures in background loops (a
Kafka publish failure, an FTP timeout — relevant from change 5 onward)
are handled by the loop's own retry logic and do not kill the runtime.

`stop()` initiates graceful shutdown. In-flight work completes (or is
cancelled) within the configured grace period. Repeated `stop()` calls
are no-ops. Repeated `start()` calls either no-op or throw
`IllegalStateException` ("already started"); the runtime never starts
duplicate workers.

In composite topology, the consuming service stops `discovery` before
`pipeline` to let the publisher drain into the internal topic before the
consumer shuts down. This is a README-documented convention, not a
library-enforced ordering.

### Validation in factories

Validation profiles defined in change 2 are wired here:

- `DiscoveryFactory.create(config)` and `.builder(config).build()` apply
  the discovery validation profile. Pipeline-only fields may be absent.
- `PipelineFactory.create(config)` and `.builder(config).build()` apply
  the pipeline validation profile. Discovery-only fields may be absent.

Validation runs at construction, not at `start()`. A misconfigured runtime
fails to construct; it never starts.

There is no composite validation profile. A consuming service running
both subsystems calls both factories; each validates its slice. The
intersection (location IDs, `internalTopicName`) is necessarily
consistent because both profiles read the same `WatcherConfig` object.

### Topic-uniqueness rules

`PipelineFactory` (which reads both `internalTopicName` and per-location
`topic`) adds two new validation rules to the pipeline profile:

1. No two locations declare the same external `topic`.
2. No location's external `topic` equals `internalTopicName`. This
   prevents the misconfiguration where pipeline accidentally publishes
   external events to the internal topic.

Discovery profile doesn't see external topics, so it doesn't enforce
these rules. The pipeline profile is the natural enforcement point.

### Public API surface assertion

A build-time test enumerates the documented public surface and verifies
the library's exposed types match. Adding a new public type without
updating the documented list fails the build.

The documented surface at the end of this change:

**Call (factories and lifecycle):**
- `DiscoveryFactory`
- `DiscoveryRuntime`
- `DiscoveryRuntimeBuilder`
- `PipelineFactory`
- `PipelineRuntime`
- `PipelineRuntimeBuilder`

**Read (configuration types):**
- `WatcherConfig`
- `LocationConfig`
- `TransportConfig` and sealed subtypes (`Direct`, `ProxyApi`, `Local`)
- `LocationSource` and sealed subtypes (`Ftp`, `ProxyApi`, `Local`)
- `FtpCredentials` and sealed subtypes (`Password`, `PrivateKey`)
- `ProxyConfig`
- `ProxyAuth`
- `DestinationConfig` (becomes sealed in the destination change)
- `OnFailureMode`
- `BackoffConfig`
- `DatabaseConfig`

**Handle (event types):**
- `FileDiscoveredEvent` (already public from change 2)

**Mount (HTTP routes):**
- `DiscoveryStatusRoutes` (already public from change 2)
- `PipelineStatusRoutes` (already public from change 2)

Everything else is `internal`. The test catches accidental exposure of
internal types (e.g. someone removes `internal` from a helper and the
build fails).

### LocalRunner migrates to the public API

`LocalRunner.discoveryOnly`, `pipelineOnly`, and `both` move from the
change-2 internal wiring to using `DiscoveryFactory.builder()` and
`PipelineFactory.builder()`. LocalRunner becomes a worked example of
how consuming services compose tests: build with overrides, start, use,
stop.

## What doesn't change

This change does **not**:

- Implement PollLoop, NotificationPublisher, the event consumer, or the
  delivery worker. The runtimes' background coroutines are skeletons at
  this point. Change 5 fills them in.
- Add new transports, destinations, or pipeline behaviour.
- Change platform-library dependencies or versions.
- Modify the configuration schema beyond the two topic-uniqueness rules
  enforced by `PipelineFactory`.
- Introduce a `WatcherFactory` or composite runtime type.

## What this means for downstream changes

- **Change 4 (`add-local-transport`)** — adds `FileTransport`,
  `RemoteFile`, `TransportException` (and subtypes) to the public surface;
  adds `withTransport(FileTransport)` to `DiscoveryRuntimeBuilder`.
  Updates `EXPECTED_PUBLIC_API`.
- **Destination change (new, after change 4)** — adds `FileDestination`,
  `WriteResult`, `DestinationException`, `DestinationRef` (sealed) to
  the public surface; adds `withDestination(FileDestination)` to
  `PipelineRuntimeBuilder`. Updates `EXPECTED_PUBLIC_API`.
- **Change 5 (`add-pipeline`)** — fills in the runtime skeletons; adds
  `FileReceivedEvent` to the public surface. `WatcherAdminRoutes` lands
  later in failure handling.

Each addition is one line in `EXPECTED_PUBLIC_API` per type. The
contract is "the surface only grows by explicit, reviewed decision."

## Impact

- Consuming services can write a minimal, production-shaped `main()`
  against the library from this change forward.
- The public API becomes a stability contract per the org's library
  versioning scheme. Future changes that break public types require a
  major version bump.
- The runtime types are constructible from this change forward, but
  inert until change 5. A consuming service can integrate the
  library's bootstrap into its own service framework (readiness check,
  status routes, lifecycle) now; the actual pipeline behaviour shows
  up when change 5 lands.
- `LocalRunner` becomes the canonical worked example of test-time
  composition. The pattern it follows is the pattern consuming services
  use for their own integration tests.

## Open questions resolved at implementation start

- The `start()` failure model — whether throwing is the right shape or
  whether a `StartResult` return type fits the platform's patterns
  better. Throwing is the proposed default; confirm.
- Grace period default for `stop()` — proposed 30 seconds; configurable.
- Build-time API-surface check approach — Kotlin metadata scanner vs.
  reflection over the loaded classes vs. a Detekt-driven rule. All
  workable; decide at implementation.
- Whether `DomainEventRouter` participates in the runtime's `stop()`
  flow automatically (via its own lifecycle hooks in `service-bootstrap`)
  or whether the runtime needs to close it explicitly. Depends on the
  platform library's contract.
