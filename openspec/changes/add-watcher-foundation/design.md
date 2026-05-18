# Design — Add Watcher Foundation

## House patterns applied

Before implementation, fetch and use the current versions of:

- `service-bootstrap` — lifecycle, config loading entry point, graceful shutdown,
  health endpoint
- `application-config` — HOCON / Typesafe Config wrapper used for typed config access
- `secret-getter` — secret retrieval component (wired now, used at runtime from
  change 5)
- `db-toolkit` — connection pool, migration runner, transaction helpers
- `observability` — structured logging (metrics and tracing are wired but not
  meaningfully exercised until later changes)

`DomainEventRouter` is NOT used in this change (no events are published yet);
it will be catalogued on-demand at change 3 when first needed.

Document the exact pinned versions in `gradle/libs.versions.toml`. Record the
versions used in the "House patterns applied" section before merging.

## Configuration schema

The watcher uses a single typed root config object loaded by `service-bootstrap`
via `ApplicationConfig` from HOCON.

```kotlin
data class WatcherConfig(
    val transport: TransportConfig,
    val storage: StorageConfig,
    val locations: List<LocationConfig> = emptyList()
)

sealed interface TransportConfig {
    data class Direct(
        val egressProxies: Map<String, ProxyConfig> = emptyMap()
    ) : TransportConfig

    data class ProxyApi(
        val baseUrl: String,
        val authSecretKey: String
    ) : TransportConfig

    data object Local : TransportConfig
}

data class StorageConfig(
    val projectId: String,
    val hostOverride: String? = null
)

data class ProxyConfig(
    val type: ProxyType,
    val host: String,
    val port: Int,
    val auth: ProxyAuth? = null
)

data class ProxyAuth(
    val username: String,                       // plain config
    val passwordSecretKey: String               // SecretGetter lookup
)

enum class ProxyType { HTTP, HTTPS, SOCKS5 }

data class LocationConfig(
    val id: String,
    val filenamePattern: String,
    val destination: DestinationConfig,
    val topic: String,
    val pollIntervalSeconds: Long,
    val onFailure: OnFailureMode,
    val maxAttempts: Int = 5,
    val backoff: BackoffConfig = BackoffConfig.default(),
    val source: LocationSource
)

sealed interface LocationSource {
    data class Ftp(
        val host: String,
        val port: Int,
        val protocol: FtpProtocol,                    // SFTP | FTPS | FTP
        val proxyName: String? = null,                // reference into Direct.egressProxies
        val credentials: FtpCredentials,
        val remotePath: String
    ) : LocationSource

    data class ProxyApi(
        val proxyLocationId: String                   // identifier on the on-prem API
    ) : LocationSource

    data class Local(
        val watchDir: String
    ) : LocationSource
}

sealed interface FtpCredentials {
    val username: String                              // plain config

    data class Password(
        override val username: String,
        val passwordSecretKey: String                 // SecretGetter lookup
    ) : FtpCredentials

    data class PrivateKey(
        override val username: String,
        val privateKeySecretKey: String,              // SecretGetter lookup
        val passphraseSecretKey: String? = null       // null when key is unencrypted
    ) : FtpCredentials
}

data class DestinationConfig(
    val bucket: String,
    val prefix: String
)

data class BackoffConfig(
    val initialSeconds: Long,
    val multiplier: Double,
    val maxSeconds: Long
) {
    companion object {
        fun default() = BackoffConfig(30, 2.0, 600)
    }
}

enum class FtpProtocol { SFTP, FTPS, FTP }
enum class OnFailureMode { HALT, SKIP_AND_ALERT }
```

The full schema is defined now even though most of it is unused in this change.
This avoids schema churn across the seven changes and lets developers and config
authors see the full surface area from the start.

### Note on the two "proxy" concepts

Two distinct concepts share the word "proxy" in the design; the schema models
them with different types:

- **`ProxyConfig`** — a generic network egress proxy (HTTP CONNECT, HTTPS
  CONNECT, SOCKS5) that tunnels TCP for a `Direct` transport's FTP/SFTP
  connection. Lives inside `TransportConfig.Direct.egressProxies`. Per-location
  use is opt-in via `LocationSource.Ftp.proxyName`.
- **`TransportConfig.ProxyApi`** — an entirely different concept: an on-prem
  HTTP API that replaces FTP, used by `ProxyApiTransport` (change 6). Not a
  network proxy in any tunnelling sense.

These never appear together in a single deployment: a watcher uses either
`Direct` (with optional egress proxies) or `ProxyApi`, never both.

## Validation rules

Validation runs at config load and fails startup on any violation. Errors
aggregate — all violations reported in one startup attempt, not one per restart.

- `transport` is required (one of `Direct`, `ProxyApi`, `Local`).
- `storage.projectId` is non-blank.
- Each `LocationConfig.id` is unique across the list.
- Each `LocationConfig.source` variant matches the active `TransportConfig`
  variant:
  - `TransportConfig.Direct` ↔ `LocationSource.Ftp`
  - `TransportConfig.ProxyApi` ↔ `LocationSource.ProxyApi`
  - `TransportConfig.Local` ↔ `LocationSource.Local`
- For `LocationSource.Ftp.proxyName`, if non-null, it MUST resolve to an entry
  in `TransportConfig.Direct.egressProxies`.
- `FtpCredentials.PrivateKey` is valid only with `FtpProtocol.SFTP`. Combinations
  with `FTPS` or `FTP` are rejected.
- `filenamePattern`, `topic`, `destination.bucket`, `destination.prefix` are
  non-blank.
- `pollIntervalSeconds >= 1`.
- `maxAttempts >= 1`.
- `backoff.initialSeconds >= 1`, `backoff.multiplier >= 1.0`,
  `backoff.maxSeconds >= initialSeconds`.

### Warnings (not errors)

- `TransportConfig.Direct.egressProxies` entry referenced by no location →
  warning at startup.

### Deferred validation (added to spec, not implemented here)

The capability spec includes a requirement that private-key credentials are
validated at startup (`SecretGetter` resolves the key, the SSH library parses
it, decryption succeeds if a passphrase is configured). This is added to the
spec in this change to establish the contract, but is not exercised here — no
`DirectTransport` exists yet (lands in change 5). Change 5's `tasks.md` will
implement this.

## Storage client construction

The GCS `Storage` client is constructed by a `StorageClientFactory` consuming
the typed `StorageConfig`:

```kotlin
class StorageClientFactory(private val config: StorageConfig) {
    fun create(): Storage =
        StorageOptions.newBuilder()
            .setProjectId(config.projectId)
            .apply { config.hostOverride?.let { setHost(it) } }
            .build()
            .service
}
```

Application code knows nothing about emulators. `hostOverride` is a generic
endpoint-override field; the deployment layer decides whether to set it.

The factory is wired and tested in this change even though no upload or download
happens yet; this proves the override mechanism works end-to-end against
`fake-gcs-server`.

## Database state table

The `ftp_file_state` table is created by a single forward migration shipped with
this change, managed by `db-toolkit`.

The full schema (see spec) is shipped now. Nullable columns for fields populated
by later state transitions (`bucket_path`, `sha256`, `last_error`) keep the
schema stable across the seven changes.

The partial index on `(location_id, status, ftp_mtime)` for non-terminal
statuses is included now because changes 3+ rely on it for the delivery worker's
pick query. Including it now lets later changes focus on logic rather than index
management.

## Health vs. status

Two endpoints because they answer two questions:

- `/health` — "is this process deployable and ready to receive traffic?"
  Returns 200 once config has loaded and migrations have applied. Used by
  Cloud Run / k8s probes.
- `/status` — "what is the watcher actually doing?" Returns operational state:
  configured locations, per-location worker state (in later changes). Used by
  humans and dashboards.

Conflating these leads to one of two failure modes: either the deploy probe
waits for operational readiness that may never come (locations halted), or
status answers are misinterpreted as deployment health.

## Local development stack

`docker-compose.yml` brings up:

- `postgres:16` with a volume for state persistence
- `fsouza/fake-gcs-server` exposing GCS on `:4443`
- Redpanda (single broker; smaller footprint than full Kafka, fast startup)
- The watcher container, with config that sets `transport.type = local` and
  watches a mounted host directory

Buckets in `fake-gcs-server` are pre-created via mounted directories under
`/storage`. Kafka topics are not created here (no publishing in this change).

A separate Compose profile (`compose --profile real-gcp`) skips the emulators
and expects the developer to provide real GCP credentials via mounted ADC files.
Day one, this profile is optional; the documented default is full-emulator.

`docs/local-development.md` covers:

- One-command startup (`docker compose up`)
- Hot reload / rebuild loop for the watcher
- How to set `platform.local.path` in `~/.gradle/gradle.properties` for
  unpublished platform-library changes (matching the existing org convention)
- How to switch to the `real-gcp` profile
- Troubleshooting: Artifactory auth, port conflicts, common HOCON parse errors

## Testing strategy

This change uses **test-name-based mapping** to satisfy the per-scenario
coverage gate in `AGENTS.md`. Each test is named to reference the scenario
it verifies, with names structured so a reader can trace from a scenario in
`spec.md` to its covering test(s) without needing a separate matrix. Examples:

- `App_boot_with_zero_locations_starts_idle`
- `Config_rejects_LocationSource_mismatched_with_TransportConfig`
- `Config_rejects_PrivateKey_credentials_with_FTPS`
- `StorageClientFactory_honours_hostOverride_when_present`
- `Migration_runs_idempotently_on_repeat_boot`

If a single test covers multiple scenarios, its name SHALL reference all of
them, or the test SHALL be split.

### Unit tests

- **Config validation** — exhaustive coverage of each validation rule. One test
  per failure mode (missing field, invalid enum, source/transport mismatch,
  unresolved proxy reference, PrivateKey with non-SFTP, etc.) plus a happy-path
  test with a fully-populated config across all three transport variants.
  Validation-error aggregation is verified (a config with multiple violations
  reports all of them).
- **Config parsing** — HOCON → `WatcherConfig` for representative configs:
  zero locations; one location per transport mode; multiple locations across
  configurations.
- **Sealed-type discriminator handling** — verify that `transport.type = direct`
  parses to `TransportConfig.Direct`, etc., and that an unknown discriminator
  value fails parsing with a clear error.
- **`StorageClientFactory`** — verifies that `hostOverride` is applied when
  present and absent when null. No actual GCS connection needed.

### Integration tests

- **App boot, fresh DB** — Testcontainers Postgres, no prior schema; service
  starts, migration runs, `ftp_file_state` table exists with the documented
  schema, `/health` returns 200.
- **App boot, idempotent migration** — Run the service twice against the same
  Testcontainers Postgres; second boot is a no-op.
- **App boot, zero locations** — Config with `locations: []`; service starts,
  `/status` reports zero locations.
- **App boot, multiple locations across modes** — One test per transport
  variant: `Direct` with `Ftp` locations, `ProxyApi` with `ProxyApi` locations,
  `Local` with `Local` locations. `/status` reports the correct count and IDs.
- **App boot, validation failure** — Config with deliberate violations
  (source/transport mismatch, unresolved proxy reference, PrivateKey on FTPS);
  service refuses to start, error output names all violations.
- **Storage client against fake-gcs-server** — `StorageClientFactory` with
  `hostOverride` set to a `fake-gcs-server` Testcontainer; verify the client
  can list buckets (trivial smoke test) to prove the override is honoured
  end-to-end.
- **Docker Compose smoke test** (optional in CI, manual otherwise) —
  `docker compose up`, wait for health, assert 200. Acceptable to gate this
  behind a CI label rather than running on every PR.

## Non-goals (deferred to later changes)

- `FileTransport` interface and implementations.
- Discovery loop, delivery worker, state transitions beyond what the schema
  describes.
- `DomainEventRouter` integration (catalogued on-demand at change 3).
- `SecretGetter` runtime usage (no secrets are resolved in this change;
  startup-time private-key validation lands in change 5).
- Metrics export beyond what `observability` provides out of the box.
- Halt-on-failure, reaper, admin endpoints.

## Risks and open questions

- **`SecretGetter` return type.** The watcher assumes `SecretGetter.get()`
  returns `String` for all secrets — passwords, passphrases, API keys, and
  private keys (in PEM/OpenSSH text form). Private keys are inherently text;
  no binary handling is needed at any boundary. If the org standard is
  `ByteArray`, the watcher decodes UTF-8 at the boundary with no functional
  difference. Confirm at implementation start.
- **Platform libraries readiness.** This change depends on `service-bootstrap`,
  `application-config`, `secret-getter`, `db-toolkit`, and `observability`
  being available. If their published versions lack features this change needs
  (e.g. HOCON sealed-type discrimination, env-var substitution in HOCON), raise
  it before working around it.
- **HOCON sealed-type discriminator.** Different libraries use different
  conventions (`type` field, parent path key, etc.). The exact convention used
  by `ApplicationConfig` should be confirmed with the platform team at
  implementation; the schema works regardless, but example configs in
  `docs/local-development.md` need to match.
- **db-toolkit migration tool.** Whatever the toolkit uses (Flyway, Liquibase,
  custom) drives the file format for the migration shipped with this change.
  Confirm at implementation start.
- **Idempotent boot under Kafka/SecretGetter unavailability.** Neither is used
  at runtime in this change. Verify that the service boots cleanly even if
  these dependencies' backends are unreachable, since the wiring exists but
  no calls are made. If platform libraries make connections eagerly at
  startup, this needs special handling.
