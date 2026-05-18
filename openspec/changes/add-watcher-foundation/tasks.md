# Tasks — Add Watcher Foundation

## 1. Resolve project facts and platform libraries

- [x] 1.1 Read `openspec/project.md`; identify `<TBD>` markers needed for this
      change (docs root, source repo, platform team contact, Artifactory URL).
- [ ] 1.2 Ask the user for the unresolved facts in a single grouped prompt;
      propose the diff to `project.md` and commit on confirmation.
- [ ] 1.3 Fetch the current `service-bootstrap`, `application-config`,
      `secret-getter`, `db-toolkit`, and `observability` library versions and
      APIs from the documented source. Do not reproduce from memory.
- [ ] 1.4 Confirm the `ApplicationConfig` convention for sealed-type
      discrimination in HOCON (discriminator field name).
- [ ] 1.5 Confirm `db-toolkit`'s migration tool (Flyway, Liquibase, or custom).
- [ ] 1.6 Record pinned versions, the discriminator convention, and any
      deviations in `design.md` under "House patterns applied."

## 2. Project skeleton

- [ ] 2.1 Initialise Gradle project with Kotlin DSL.
- [ ] 2.2 Create `gradle/libs.versions.toml` with pinned platform-library versions.
- [ ] 2.3 Configure `settings.gradle.kts` with the `platform.local.path`
      composite-build switch, matching the existing org convention.
- [ ] 2.4 Add `build.gradle.kts` with dependencies on `service-bootstrap`,
      `application-config`, `secret-getter`, `db-toolkit`, `observability`.
- [ ] 2.5 Set up Detekt / ktlint per org convention (if applicable).

## 3. Configuration schema

- [ ] 3.1 Define `WatcherConfig` and the sealed `TransportConfig`,
      `LocationSource`, and `FtpCredentials` hierarchies per `design.md`.
- [ ] 3.2 Define `ProxyConfig`, `ProxyAuth`, `StorageConfig`, `DestinationConfig`,
      `BackoffConfig`, and the supporting enums.
- [ ] 3.3 Implement validation rules per `design.md`. Aggregate violations.
- [ ] 3.4 Wire config loading through `service-bootstrap` + `application-config`.
- [ ] 3.5 Build `StorageClientFactory` consuming `StorageConfig`; verify
      `hostOverride` is honoured.

## 4. Database state table

- [ ] 4.1 Write the migration for `ftp_file_state` per the schema in `spec.md`,
      using the migration tool provided by `db-toolkit`.
- [ ] 4.2 Include the partial index on non-terminal statuses.
- [ ] 4.3 Wire migration execution into service startup; failure SHALL fail
      startup.

## 5. Health and status endpoints

- [ ] 5.1 Expose `/health` via `service-bootstrap` (likely already provided;
      configure).
- [ ] 5.2 Implement `/status` returning the configured-locations summary.
- [ ] 5.3 Ensure `/health` returns 200 only after config load and migration
      completion.

## 6. Local development stack

- [ ] 6.1 Write `docker-compose.yml` with Postgres, `fake-gcs-server`, and
      Redpanda.
- [ ] 6.2 Pre-create the expected dev bucket in `fake-gcs-server`
      (mounted directory).
- [ ] 6.3 Add a `real-gcp` Compose profile that omits emulators and expects ADC.
- [ ] 6.4 Write `docs/local-development.md` per the spec.
- [ ] 6.5 Verify `docker compose up` brings up the stack and the watcher reaches
      `/health` 200.

## 7. Testing — unit

- [ ] 7.1 Validation rules: one test per failure mode (structural, sealed-type
      mismatch, unresolved proxy reference, PrivateKey on non-SFTP, etc.),
      plus a fully-populated happy-path case per transport variant. Verify
      error aggregation.
- [ ] 7.2 Config parsing: HOCON → `WatcherConfig` for representative configs
      (zero locations, one per transport mode, multiple locations).
- [ ] 7.3 Sealed-type discriminator parsing: each variant parses correctly;
      unknown discriminator value fails with a clear error.
- [ ] 7.4 `StorageClientFactory`: behaviour with and without `hostOverride`.

## 8. Testing — integration

- [ ] 8.1 App boot against fresh Testcontainers Postgres; migration runs;
      `ftp_file_state` has the documented schema; `/health` is 200.
- [ ] 8.2 App boot is idempotent across two consecutive starts against the
      same DB.
- [ ] 8.3 App boot with `locations: []`; `/status` reports zero locations.
- [ ] 8.4 App boot per transport variant (`Direct` with `Ftp` locations,
      `ProxyApi` with `ProxyApi` locations, `Local` with `Local` locations);
      `/status` reports correct count and IDs in each case.
- [ ] 8.5 App boot rejects configurations with deliberate violations
      (source/transport mismatch, unresolved proxy reference, PrivateKey on
      FTPS); error output names all violations.
- [ ] 8.6 `StorageClientFactory` against `fake-gcs-server` Testcontainer;
      verify the override is honoured by a list-buckets smoke test.
- [ ] 8.7 (Optional, CI-labelled) Docker Compose smoke test: `docker compose
      up`, wait for health, assert 200.

## 9. Documentation

- [ ] 9.1 `README.md` with one-paragraph service description, link to
      `docs/local-development.md`, link to OpenSpec change folder.
- [ ] 9.2 Confirm `docs/local-development.md` covers all items listed in
      `design.md`.

## 10. Pre-merge checklist

- [ ] 10.1 All `<TBD>` markers in `project.md` that this change depends on are
      resolved.
- [ ] 10.2 `design.md` "House patterns applied" section names the exact
      platform-library versions used and the confirmed HOCON discriminator
      convention.
- [ ] 10.3 Unit tests pass in CI.
- [ ] 10.4 Integration tests pass in CI.
- [ ] 10.5 Every scenario in `spec.md` added by this change is covered by at
      least one named test. Mapping is either test-name-based or documented
      as a coverage matrix in `design.md` (one approach chosen and applied
      consistently).
- [ ] 10.6 No env-var or system-property reads in application code (grep for
      `System.getenv` and `System.getProperty` outside `service-bootstrap` /
      `application-config` integration; should be zero).
- [ ] 10.7 No emulator-awareness in application code (grep for `emulator`,
      `fake-gcs`, `localhost`; only acceptable in test fixtures, Compose
      files, and `docs/`).
- [ ] 10.8 No raw secret values in any config file (grep for `password:`,
      `private_key:`; only `*SecretKey` fields should appear).
