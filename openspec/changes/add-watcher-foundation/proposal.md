# Add Watcher Foundation

## Why

`svc-ftp-location-watcher` is a new service in the ecosystem. Before any watching,
polling, downloading, or publishing logic can be built, the service needs a
foundation: a bootable application, a typed configuration schema, a database
state table, a local development stack, and the wiring that ties them together.

This change delivers that foundation. The service after this change boots, validates
its configuration, runs database migrations, and exposes a health endpoint — but
does no FTP work. A configuration with zero watched locations is valid; the service
idles. This is the "deployable but disabled" state the deployment strategy depends
on.

Subsequent changes will add the `FileTransport` abstraction, the discovery and
delivery pipeline, failure handling, real transports, and observability — each
building on this foundation.

## What changes

- Gradle project skeleton with version catalogue and the org-standard
  `platform.local.path` composite-build switch for unpublished platform-library
  changes.
- Typed configuration schema (Kotlin data classes) covering the watcher's full
  intended config surface, loaded via the platform-standard `ApplicationConfig`
  (HOCON / Typesafe Config). Includes:
  - A sealed `TransportConfig` hierarchy (`Direct`, `ProxyApi`, `Local`).
  - A sealed `LocationSource` hierarchy that mirrors `TransportConfig`.
  - A sealed `FtpCredentials` hierarchy (`Password`, `PrivateKey`).
  - A top-level `egressProxies` registry of named `ProxyConfig` entries inside
    `TransportConfig.Direct`, referenced by `LocationSource.Ftp.proxyName`.
  - `ProxyAuth` as a nested optional unit encapsulating co-required credentials.
  - The `SecretKey` suffix convention for `SecretGetter` lookup fields.
- Database state table `ftp_file_state` defined via a `db-toolkit` migration,
  shipped complete (full schema for all later changes) to avoid follow-up
  migrations.
- Docker Compose dev stack: Postgres, `fake-gcs-server`, Redpanda.
- Service bootstrap wired via the `service-bootstrap` library: lifecycle, config
  loading, graceful shutdown, structured logging, health endpoint.
- `/health` (deployability signal — config loaded, migrations applied) and
  `/status` (operational signal — populated by later changes; reports
  configured-locations summary for now).
- Unit tests for config validation and schema parsing. Integration test for app
  boot against Testcontainers Postgres with migrations applied.
- `docs/local-development.md` describing the dev stack and the
  `platform.local.path` mechanism.

## What does not change

This change does **not**:

- Implement the `FileTransport` interface or any transport.
- Implement discovery, delivery, download, or publish logic.
- Resolve any secrets at runtime (the `SecretGetter` dependency is wired but
  no secret reads happen yet; no `FtpCredentials` are exercised).
- Connect to Kafka at runtime (no events are published; `DomainEventRouter`
  is not yet a dependency).
- Connect to GCS at runtime (the `StorageClientFactory` is built and tested
  for the `hostOverride` behaviour, but no uploads occur).
- Define per-transport behaviour beyond schema validation.

Those are the subject of later changes.

## Impact

- New service skeleton in the repo. Deployable to Cloud Run or as a container
  on-prem.
- New database (or schema) hosting `ftp_file_state`. Provisioning of the DB
  itself is an infra concern outside this change; the migration is in scope.
- No Kafka topics produced or consumed yet.
- No production rollout: this change is deployable but the service does nothing
  useful until later changes land.
