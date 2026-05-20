# Design — Add Library Public API

## House patterns applied

No new platform-library dependencies. The factories, runtimes, and
builders wire types and validators that already exist from changes 1
and 2.

Confirm at implementation start:

- The platform's preferred build-time API-surface check approach
  (Kotlin metadata scanner, reflection-over-loaded-classes, or a Detekt
  rule).
- Whether `DomainEventRouter` participates in `service-bootstrap`'s
  shutdown lifecycle automatically, or whether each `Runtime.stop()`
  must close it explicitly.
- Default grace period for `stop()` (proposed: 30 seconds).

## Public API surface

The complete documented surface at the end of this change:

### Call (factories and lifecycle)

- `DiscoveryFactory` — object with `create(WatcherConfig)` and
  `builder(WatcherConfig)`
- `DiscoveryRuntime` — class with `start()`, `stop()`, `isRunning()`
- `DiscoveryRuntimeBuilder` — class with `withSecretGetter`,
  `withClock`, `withDomainEventRouter`, `build()`
- `PipelineFactory` — object with `create(WatcherConfig)` and
  `builder(WatcherConfig)`
- `PipelineRuntime` — class with `start()`, `stop()`, `isRunning()`
- `PipelineRuntimeBuilder` — class with `withSecretGetter`, `withClock`,
  `withDomainEventRouter`, `build()`

### Read (configuration types — already public from changes 1-2)

- `WatcherConfig`
- `LocationConfig`
- `TransportConfig` (sealed: `Direct`, `ProxyApi`, `Local`)
- `LocationSource` (sealed: `Ftp`, `ProxyApi`, `Local`)
- `FtpCredentials` (sealed: `Password`, `PrivateKey`)
- `ProxyConfig`, `ProxyAuth`
- `DestinationConfig` (becomes sealed in the destination change)
- `OnFailureMode`, `BackoffConfig`, `DatabaseConfig`

### Handle (events — already public from change 2)

- `FileDiscoveredEvent`

### Mount (routes — already public from change 2)

- `DiscoveryStatusRoutes`
- `PipelineStatusRoutes`

### Added in later changes (NOT in this change's surface)

- `FileTransport`, `RemoteFile`, `TransportException` — change 4
- `withTransport` on `DiscoveryRuntimeBuilder` — change 4
- `FileDestination`, `WriteResult`, `DestinationException` — destination change
- `DestinationRef` (sealed) — destination change
- `withDestination` on `PipelineRuntimeBuilder` — destination change
- `FileReceivedEvent` — change 5
- `WatcherAdminRoutes` — change 6 (failure handling)

Each of these is added to `EXPECTED_PUBLIC_API` by its own change, with
the surface test updating to enforce the new floor.

## Two construction paths

Both factories expose two paths for the same reason:

**`create(config)` — production.** The 99% case. Consuming service reads
config from `ApplicationConfig`, hands it to `create`, gets a runtime.
No knobs. No surprises.

**`builder(config)` — tests and special-case deployments.** Allows
component overrides. Two real use cases:

1. **Test fixtures** — `LocalRunner` and consuming services' integration
   tests inject `StubSecretGetter`, `InMemoryDomainEventRouter`, fixed
   clocks. Same wiring as production, different leaves.
2. **Special-case deployments** — a consumer that needs a custom
   `DomainEventRouter` (e.g. wrapped with extra observability) can
   inject it without modifying the library.

Both paths flow through the same internal assembler. The factory just
chooses whether to expose the builder's override hooks before
delegating. This guarantees equivalent validation and equivalent wiring
between the paths.

```
DiscoveryFactory.create(config)
    ─▶ DiscoveryRuntimeBuilder(config).build()

DiscoveryFactory.builder(config)
    ─▶ DiscoveryRuntimeBuilder(config)   (returned to caller)

DiscoveryRuntimeBuilder.build()
    ─▶ DiscoveryRuntimeAssembler.assemble(config, overrides)
    ─▶ new DiscoveryRuntime(...)
```

Tests verify that `create()` and `builder().build()` produce equivalent
runtimes for the same config (modulo overrides).

## Builder override surface

Both builders share three overrides at this change:

- `withSecretGetter(SecretGetter)` — covers test secret stores; covers
  consumers that wrap the platform `SecretGetter` with caching or
  observability.
- `withClock(Clock)` — covers deterministic time in tests; covers
  consumers running against a logical clock for replay scenarios.
- `withDomainEventRouter(DomainEventRouter)` — covers
  `InMemoryDomainEventRouter` for unit tests; covers consumers that
  wrap the router with cross-cutting concerns.

We considered exposing `withDatabasePool` (or similar) to allow custom
DB pools. **Rejected.** The pool is built from `WatcherConfig.db.*`;
overriding it would let tests skip migrations or mask the connection
contract. Tests that want in-memory DB use Testcontainers; the override
isn't needed.

We considered exposing the readiness check or status routes as
overridable components. **Rejected.** The library *contributes* a
readiness check and *exposes* status routes; mounting and bridging
those to the platform is the consuming service's concern, owned by the
consumer's HTTP framework configuration. Adding builder hooks here
would conflate library and consumer responsibilities.

Each addition to the override surface in later changes follows the same
test: does the abstraction have meaningful test substitutes that
consumers would reasonably swap? If yes, it earns an override. If no,
it stays wired-from-config only.

## Lifecycle semantics

### `start()` — non-blocking, throws on synchronous startup failure

```
start() {
    // Synchronous: must succeed for the runtime to be considered started
    applyMigrations()           // DB migration apply
    openConnectionPool()        // db-toolkit pool open
    createKafkaClient()         // DomainEventRouter init
    subscribeIfPipeline()       // pipeline-only: consumer subscription
    contributeReadinessReady()  // tell service-bootstrap we're up

    // Asynchronous: launched and forgotten (managed by scope)
    launchBackgroundCoroutines()  // pollLoops, publishers, workers — empty at change 3
}
```

If any synchronous step fails, `start()` throws and the runtime is in
a "never started" state. `isRunning()` returns false; `stop()` is a
no-op.

Why throw, not return a result type? Three reasons:

1. **Kotlin / JVM convention.** Constructors and initialisers throw on
   failure. A factory that returns a result type for startup failure is
   surprising.
2. **`main()` simplicity.** The consuming service's `main()` calls
   `runtime.start()` once; catching an exception there is natural. A
   result type would require explicit unwrapping at every call site.
3. **Composability.** Coroutine scopes, structured concurrency, and
   try/catch all compose well with throwing initialisation. Result
   types fight all three.

### `start()` is non-blocking

`start()` returns once synchronous work completes. Background coroutines
continue running in the runtime's owned scope. The consuming service's
`main()` then joins on its own signal (typically a shutdown hook calling
`stop()`, plus an `awaitShutdown()` mechanism the platform provides).

This matches `LocalRunner`'s shape and how production consuming services
will work. The alternative — `start()` blocks until `stop()` is called
externally — requires the consumer to manage threading more carefully
and reduces flexibility.

### Synchronous-startup failures are real failures

Examples of failures that throw:

- DB is unreachable → `db-toolkit` throws during pool init.
- Migration fails to apply → migration runner throws.
- Kafka broker is unreachable → producer init throws (depending on the
  client; some defer this to first publish, in which case it's not a
  synchronous failure).
- Configuration is invalid in a way validation didn't catch (rare; the
  validation profiles should be comprehensive enough).

These are all "we cannot meaningfully run." Throwing is the right shape.

### `stop()` — graceful, idempotent

```
stop() {
    if (!running.compareAndSet(true, false)) return   // already stopped
    cancelBackgroundCoroutines()                       // signal cancellation
    awaitWithTimeout(gracePeriod)                      // wait for clean shutdown
    closeKafkaClient()
    closeConnectionPool()
    // status routes module remains queryable; readiness reports NOT_READY
}
```

After the grace period elapses, in-flight work is cancelled hard.
Connection pools close. The runtime is now genuinely stopped — calling
`start()` again would re-do the synchronous init, which probably
won't work (migrations might fail on re-init, depending on `db-toolkit`).

We do not support `start()` after `stop()`. If a consuming service
wants to restart, it constructs a new runtime. The factories are cheap
to call.

### Composite stop ordering

`LocalRunner.both` stops discovery before pipeline. Consuming services
running both subsystems are documented to do the same. The reason:

- Discovery's NotificationPublisher publishes `FileDiscoveredEvent` to
  the internal topic.
- Pipeline's consumer subscribes to the internal topic.
- If pipeline stops first, discovery may publish events that have no
  consumer until pipeline restarts. Those events sit in Kafka — not
  lost, but the latency to processing grows.
- If discovery stops first, it stops publishing immediately. Pipeline
  can drain the topic and process the last batch of events before
  shutting down.

This is a convention, not enforcement. A consumer that wants different
ordering (e.g. shutdown coordination during a deploy where pipeline
restarts faster than discovery) is free to choose. The README's
"Composite topology" section documents the recommended order.

## Validation invocation

The factories wire change 2's validation profiles:

```
object DiscoveryFactory {
    fun create(config: WatcherConfig): DiscoveryRuntime =
        builder(config).build()

    fun builder(config: WatcherConfig): DiscoveryRuntimeBuilder {
        return DiscoveryRuntimeBuilder(config)
    }
}

class DiscoveryRuntimeBuilder internal constructor(
    private val config: WatcherConfig,
) {
    private var secretGetter: SecretGetter? = null
    private var clock: Clock? = null
    private var router: DomainEventRouter? = null

    fun withSecretGetter(s: SecretGetter): DiscoveryRuntimeBuilder = apply { secretGetter = s }
    fun withClock(c: Clock): DiscoveryRuntimeBuilder = apply { clock = c }
    fun withDomainEventRouter(r: DomainEventRouter): DiscoveryRuntimeBuilder = apply { router = r }

    fun build(): DiscoveryRuntime {
        val errors = ConfigValidator.validateForDiscovery(config)
        if (errors.hasErrors) throw ConfigValidationException(errors)
        return DiscoveryRuntimeAssembler.assemble(
            config = config,
            overrides = Overrides(secretGetter, clock, router),
        )
    }
}
```

`PipelineFactory` is symmetric, except `build()` runs both the pipeline
profile and the two topic-uniqueness rules added in this change.
Validation aggregates errors across all rules; one validation pass per
build.

## Topic-uniqueness rules

Two new rules added by `PipelineFactory.build()`:

```kotlin
// In PipelineRuntimeBuilder.build(), after the pipeline profile:

val topicErrors = mutableListOf<ValidationError>()

// Rule 1: no two locations declare the same external topic
config.locations
    .groupBy { it.topic }
    .filter { (_, locs) -> locs.size > 1 }
    .forEach { (topic, locs) ->
        topicErrors += ValidationError(
            "duplicate external topic '$topic' used by locations: " +
            locs.joinToString { it.id }
        )
    }

// Rule 2: no location's topic equals internalTopicName
config.locations
    .filter { it.topic == config.internalTopicName }
    .forEach { loc ->
        topicErrors += ValidationError(
            "location '${loc.id}' declares topic '${loc.topic}' which " +
            "clashes with internalTopicName"
        )
    }
```

`DiscoveryFactory.build()` does not run these rules because discovery
doesn't read external `topic` values. Discovery profile validation
ignores those fields, and the topic-uniqueness rules don't fire.

If a consumer in composite topology builds both factories, the pipeline
factory enforces the rules. Discovery's validation passing first
doesn't mask pipeline's later failure — they're independent calls, each
runs its own validators.

## Public API surface enforcement

```kotlin
class PublicApiSurfaceTest {

    private val expectedPublicApi: Set<String> = setOf(
        "com.example.ftpwatcher.discovery.DiscoveryFactory",
        "com.example.ftpwatcher.discovery.DiscoveryRuntime",
        "com.example.ftpwatcher.discovery.DiscoveryRuntimeBuilder",
        "com.example.ftpwatcher.pipeline.PipelineFactory",
        "com.example.ftpwatcher.pipeline.PipelineRuntime",
        "com.example.ftpwatcher.pipeline.PipelineRuntimeBuilder",
        "com.example.ftpwatcher.shared.config.WatcherConfig",
        "com.example.ftpwatcher.shared.config.LocationConfig",
        // ... full list
    )

    @Test
    fun `public api surface matches expected`() {
        val actual = scanLibraryPublicTypes()
        assertThat(actual).containsExactlyInAnyOrder(*expectedPublicApi.toTypedArray())
    }
}
```

`scanLibraryPublicTypes` reads the compiled library jar (the `main`
configuration's output, not testFixtures) and enumerates types whose
Kotlin visibility is `PUBLIC`. Sealed subtypes count as separate
entries; nested classes count individually.

Adding a new public type without updating `expectedPublicApi` fails
this test. Removing one fails it too. The list is the contract.

Three possible implementations:

1. **Kotlin metadata scanner** (e.g. `kotlin-metadata-jvm`) — reads
   `@Metadata` annotations on classes. Most accurate; minor compiler-
   version sensitivity.
2. **Java reflection** — load each class with `ClassLoader.loadClass`,
   check `Modifier.PUBLIC` on classes. Works but doesn't distinguish
   Kotlin `internal` from Java `public` reliably.
3. **Detekt custom rule** — at-source check. Different timing (compile
   vs. test), same coverage.

Decision deferred to implementation; (1) is the proposed default.

## LocalRunner migration

`LocalRunner` in change 2 constructs runtimes via an internal initialiser
(documented as "to be replaced by public factories in change 3"). This
change replaces it:

```kotlin
class LocalRunner private constructor(
    private val discovery: DiscoveryRuntime?,
    private val pipeline: PipelineRuntime?,
) {
    fun start() {
        discovery?.start()
        pipeline?.start()
    }

    fun close() {
        discovery?.stop()
        pipeline?.stop()
    }

    companion object {
        fun discoveryOnly(configPath: String): LocalRunner =
            LocalRunner(
                discovery = DiscoveryFactory.builder(loadConfig(configPath))
                    .withSecretGetter(StubSecretGetter.fromEnv())
                    .build(),
                pipeline = null,
            )

        fun pipelineOnly(configPath: String): LocalRunner =
            LocalRunner(
                discovery = null,
                pipeline = PipelineFactory.builder(loadConfig(configPath))
                    .withSecretGetter(StubSecretGetter.fromEnv())
                    .build(),
            )

        fun both(configPath: String): LocalRunner {
            val config = loadConfig(configPath)
            val secrets = StubSecretGetter.fromEnv()
            return LocalRunner(
                discovery = DiscoveryFactory.builder(config)
                    .withSecretGetter(secrets)
                    .build(),
                pipeline = PipelineFactory.builder(config)
                    .withSecretGetter(secrets)
                    .build(),
            )
        }
    }
}
```

`LocalRunner` now exercises the public API end-to-end. A consuming
service's integration tests can mirror this pattern directly:
`Factory.builder(...).withSomething(...).build()`, start, exercise,
stop.

## Testing strategy

Test-name-based mapping per `AGENTS.md`. Most scenarios in `spec.md`
map directly to test names.

### Unit tests

- `DiscoveryFactory_create_returns_runnable_runtime_for_valid_config`
- `DiscoveryFactory_create_aggregates_validation_errors_for_invalid_config`
- `DiscoveryFactory_create_tolerates_absent_pipeline_fields`
- `DiscoveryFactory_builder_allows_secretGetter_override`
- `DiscoveryFactory_builder_allows_clock_override`
- `DiscoveryFactory_builder_allows_domainEventRouter_override`
- `DiscoveryFactory_builder_rejects_invalid_config_at_build`
- `DiscoveryFactory_create_and_builder_produce_equivalent_runtimes_for_same_config`
- `DiscoveryRuntime_start_then_stop_clean_lifecycle`
- `DiscoveryRuntime_synchronous_startup_failure_throws`
- `DiscoveryRuntime_double_start_is_idempotent_or_throws`
- `DiscoveryRuntime_double_stop_is_noop`
- `DiscoveryRuntime_isRunning_reflects_state`
- (Parallel set for `PipelineFactory` / `PipelineRuntime` / `PipelineRuntimeBuilder`)
- `PipelineRuntimeBuilder_does_not_expose_withTransport`
- `PipelineFactory_rejects_duplicate_external_topics_across_locations`
- `PipelineFactory_rejects_location_topic_matching_internalTopicName`
- `PipelineFactory_tolerates_absent_discovery_fields`
- `DiscoveryFactory_ignores_external_topic_uniqueness_rules`
- `Public_api_surface_matches_expected`
- `Adding_undocumented_public_type_fails_surface_test`
- `Composite_validation_profile_is_removed_from_codebase`

### Integration tests

- `LocalRunner_uses_public_factories_for_construction`
- `LocalRunner_both_stops_discovery_before_pipeline`
- `Consumer_main_pattern_starts_discovery_runtime` — small in-process
  consumer mimicking the documented discovery-only `main()` pattern
  boots against Testcontainers DB.
- `Consumer_main_pattern_starts_pipeline_runtime` — same for pipeline.
- `Consumer_main_pattern_starts_both_runtimes` — composite mode.

## Risks and open questions

- **API surface test approach.** Kotlin metadata scanners can be
  finicky across compiler versions. The fallback (reflection over
  `PUBLIC` modifier) catches less but never breaks. Decide at
  implementation.
- **`start()` throw vs. return result.** Strongly recommended throw,
  per the rationale above. If the platform's other libraries use result
  types for startup, consistency might justify changing this — confirm
  at implementation.
- **`DomainEventRouter` shutdown semantics.** If the platform's router
  manages its own lifecycle via `service-bootstrap`, the runtime's
  `stop()` doesn't close it explicitly. If not, `stop()` closes it
  after cancelling background coroutines. Either works; the difference
  affects only the implementation, not the spec.
- **`start()` after `stop()`.** We document this as unsupported (build
  a new runtime). The factories are cheap. If consumers find this
  awkward, a later change could add restart support; for now, the
  simpler contract.
