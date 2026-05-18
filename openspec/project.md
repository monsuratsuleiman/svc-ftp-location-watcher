# Project: svc-ftp-location-watcher

A reusable service that watches configured FTP/SFTP locations (or an on-prem HTTP API
that fronts them), downloads new files, streams them directly into Google Cloud Storage,
and publishes "file received" events. Consumers subscribe to the per-location event
topic and process landed files.

The watcher is one component in a larger ecosystem of services that share platform
libraries for bootstrap, config, secrets, persistence, event routing, and observability.

## Tech stack

- **Language:** Kotlin (JVM 21+)
- **Build:** Gradle with Kotlin DSL, version catalogue (`gradle/libs.versions.toml`)
- **Runtime:** Containerised; deploys to Cloud Run (GCP) or as a container on-prem
- **Database:** PostgreSQL (Cloud SQL in production, container locally)
- **Messaging:** Apache Kafka, via the org-standard `DomainEventRouter` component
  (catalogued on-demand when first needed)
- **Object storage:** Google Cloud Storage (real in production; `fake-gcs-server`
  locally)
- **Config:** HOCON via Typesafe Config, accessed through `ApplicationConfig`
- **Local dev:** Docker Compose with emulated GCS and Kafka

## Platform libraries

The org provides versioned Gradle libraries for cross-cutting concerns. This service
depends on them; it does not reimplement them.

| Library | Purpose | Coordinates |
|---|---|---|
| `service-bootstrap` | Service skeleton, lifecycle, shutdown, health endpoints | `com.yourorg.platform:service-bootstrap` |
| `application-config` | HOCON config loading via Typesafe Config; the standard config-access API | `com.yourorg.platform:application-config` |
| `secret-getter` | The standard secret-retrieval component. All secrets resolved through this — no direct Secret Manager SDK use | `com.yourorg.platform:secret-getter` |
| `db-toolkit` | Connection pooling, migration tooling, transaction helpers | `com.yourorg.platform:db-toolkit` |
| `observability` | Metrics, tracing, structured logging conventions | `com.yourorg.platform:observability` |

Pinned versions live in `gradle/libs.versions.toml`.

**This catalogue is partial.** It lists what this watcher knows it uses today.
The org maintains other platform libraries (e.g. for event routing, retry strategies,
HTTP clients) that this service may need as it grows. When a change touches a
cross-cutting concern not listed here, follow the procedure in `AGENTS.md`
("Platform-library gate") to ask the user about org-standard libraries before
implementing.

- **Docs / API reference:** `<TBD>`
- **Source repo:** `<TBD>`
- **Owner / contact:** `<TBD>`

### Local development against unpublished platform changes

The standard mechanism is Gradle composite builds via `includeBuild`. Set
`platform.local.path` (in `~/.gradle/gradle.properties` or via `-P`) to a local
checkout of the platform repo; Gradle substitutes published artifacts with local
projects.

This pattern is already used in other services in the org; match the existing
convention rather than inventing a new one. See `docs/local-development.md`
(created by the foundation change) for setup steps.

## Conventions

- **Naming:** Services are `svc-<name>`. Kafka topics are dot-separated:
  `ftp.<location-id>.received`. DB tables use `snake_case`.
- **Config format:** HOCON. Config keys are `camelCase`, matching the Kotlin
  data-class property names they bind to. Application code reads typed config
  objects loaded via `ApplicationConfig`. No direct env-var reads in application
  code.
- **HTTP JSON bodies:** request and response bodies use `camelCase` keys
  (e.g. `/status` returns `locationsConfigured`), consistent with the Kotlin
  property names and the config convention. Set the JSON mapper's naming
  strategy once at the service boundary; do not hand-name fields per endpoint.
- **Endpoint overrides:** emulator/local endpoints are passed via typed config
  fields (e.g. `hostOverride` on storage config), not via SDK-specific environment
  variables. Production code does not know that emulators exist.
- **Secrets:** Loaded only via `SecretGetter`. Never inline, never logged.
  Config fields holding `SecretGetter` lookup keys use the `SecretKey` suffix
  (e.g. `passwordSecretKey`).
- **Tests:** Unit tests for logic, integration tests for cross-component or
  external-system interaction. Testcontainers for Postgres, `fake-gcs-server` for
  GCS, Redpanda (or single-broker Kafka) for Kafka, Apache MINA SSHD for SFTP,
  WireMock for HTTP proxies.
- **Migrations:** Provided by `db-toolkit`. Migrations run at startup; failure
  to migrate fails service start.

## Org infrastructure

- **Artifact registry (Maven):** `<TBD>` — JFrog Artifactory; credentials configured
  per developer per the platform-team setup guide.
- **Internal docs root:** `<TBD>`
- **Platform team contact:** `<TBD>`

## Unresolved facts

Entries marked `<TBD>` above are unresolved. When implementing a change that needs
one, follow the procedure in `AGENTS.md` ("Resolving unresolved project facts") to
ask the user, then commit the answer to this file.
