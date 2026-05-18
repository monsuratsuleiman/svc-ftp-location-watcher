# ftp-watcher Capability — Delta

## ADDED Requirements

### Requirement: Service boots with a valid configuration

The watcher SHALL load and validate its configuration at startup. Invalid
configuration SHALL prevent startup with a clear error indicating the
offending field. Validation errors SHALL aggregate — report all violations
at once, not one per restart.

#### Scenario: Valid configuration with zero locations

- **GIVEN** a configuration file with `locations: []` (empty list)
- **WHEN** the service starts
- **THEN** startup succeeds
- **AND** the service logs "no locations configured, idling"
- **AND** `/health` returns 200 OK

#### Scenario: Valid configuration with one or more locations

- **GIVEN** a configuration file with one or more well-formed location entries
- **WHEN** the service starts
- **THEN** startup succeeds
- **AND** each location's configuration is parsed into a typed `LocationConfig`
- **AND** `/health` returns 200 OK

#### Scenario: Invalid configuration (missing required field)

- **GIVEN** a configuration file with a location missing a required field
- **WHEN** the service starts
- **THEN** startup fails with a non-zero exit code
- **AND** the log message names the offending location id and field
- **AND** any other validation errors in the same config are also reported

### Requirement: Configuration uses ApplicationConfig and typed objects

The watcher SHALL define its configuration as Kotlin data classes loaded via the
org-standard `ApplicationConfig` (HOCON / Typesafe Config). Application code
SHALL NOT read environment variables or system properties directly; all
external inputs flow through the typed config object.

#### Scenario: Storage host override configured via config field

- **GIVEN** a configuration with `storage.hostOverride` set to a non-null URL
- **WHEN** the watcher constructs its GCS client
- **THEN** the client is configured with the override host
- **AND** application code does not consult `STORAGE_EMULATOR_HOST` or any other
  SDK-specific environment variable

#### Scenario: Storage host override absent

- **GIVEN** a configuration without `storage.hostOverride`
- **WHEN** the watcher constructs its GCS client
- **THEN** the client uses the GCS SDK's default endpoint

### Requirement: Configuration schema models transport variants as a sealed hierarchy

The watcher's transport mode SHALL be modelled as a sealed `TransportConfig`
hierarchy with three variants: `Direct`, `ProxyApi`, and `Local`. Each variant
SHALL carry only the configuration its corresponding transport requires.
A separate `mode` enum SHALL NOT exist; the variant *is* the mode.

#### Scenario: Direct transport carries egress proxy registry

- **GIVEN** a configuration with `transport.type = direct`
- **WHEN** the configuration is parsed
- **THEN** the resulting `TransportConfig` is the `Direct` variant
- **AND** the variant carries a (possibly empty) map of named egress proxies

#### Scenario: ProxyApi transport carries the FTP-API endpoint

- **GIVEN** a configuration with `transport.type = proxyApi`
- **WHEN** the configuration is parsed
- **THEN** the resulting `TransportConfig` is the `ProxyApi` variant
- **AND** the variant carries `baseUrl` and `authSecretKey`

#### Scenario: Local transport has no transport-specific config

- **GIVEN** a configuration with `transport.type = local`
- **WHEN** the configuration is parsed
- **THEN** the resulting `TransportConfig` is the `Local` variant (no fields)

### Requirement: Location source variant matches transport variant

Each `LocationConfig.source` SHALL be a sealed `LocationSource` variant
(`Ftp`, `ProxyApi`, or `Local`) that matches the active `TransportConfig`
variant. A mismatch SHALL fail startup validation.

#### Scenario: Direct transport with Ftp location source

- **GIVEN** `transport.type = direct` and a location with `source.type = ftp`
- **WHEN** the configuration is validated
- **THEN** validation passes

#### Scenario: Direct transport with mismatched location source

- **GIVEN** `transport.type = direct` and a location with `source.type = local`
- **WHEN** the configuration is validated
- **THEN** validation fails
- **AND** the error names the offending location id and the mismatch

### Requirement: Egress proxies are defined once and referenced by name

For `TransportConfig.Direct`, network egress proxies SHALL be defined once in
the top-level `egressProxies: Map<String, ProxyConfig>` registry, keyed by a
name. Each `LocationSource.Ftp` MAY reference one of these proxies by its
`proxyName: String?` field. A null `proxyName` means a direct network
connection (no tunnelling). Every non-null `proxyName` SHALL resolve to an
entry in the registry.

The `ProxyConfig` type is generic (HTTP / HTTPS / SOCKS5) and not FTP-specific.

#### Scenario: Location with proxy reference

- **GIVEN** `transport.egressProxies` contains an entry `corp-socks`
- **AND** a location with `source.proxyName = corp-socks`
- **WHEN** the configuration is validated
- **THEN** validation passes

#### Scenario: Location with unresolved proxy reference

- **GIVEN** `transport.egressProxies` contains no entry `corp-socks`
- **AND** a location with `source.proxyName = corp-socks`
- **WHEN** the configuration is validated
- **THEN** validation fails
- **AND** the error names the offending location and the unresolved name

#### Scenario: Location with direct connection (no proxy reference)

- **GIVEN** a location with `source.proxyName = null` (field absent)
- **WHEN** the configuration is validated
- **THEN** validation passes

#### Scenario: Unused egress proxy

- **GIVEN** `transport.egressProxies` contains an entry referenced by no location
- **WHEN** the configuration is validated
- **THEN** validation passes
- **AND** a warning is logged naming the unused proxy

### Requirement: Proxy auth is encapsulated as a nested optional unit

`ProxyConfig.auth` SHALL be a nullable `ProxyAuth` block. When non-null, the
block SHALL contain both `username` (plain config) and `passwordSecretKey` (a
`SecretGetter` lookup key) — neither field individually nullable. A null
`auth` block means the proxy requires no authentication.

#### Scenario: Proxy with auth

- **GIVEN** a proxy entry with `auth.username` and `auth.passwordSecretKey` set
- **WHEN** the configuration is parsed
- **THEN** validation passes
- **AND** the resulting `ProxyConfig.auth` is non-null

#### Scenario: Proxy without auth

- **GIVEN** a proxy entry with no `auth` block
- **WHEN** the configuration is parsed
- **THEN** validation passes
- **AND** the resulting `ProxyConfig.auth` is null

### Requirement: FTP credentials are modelled as a sealed hierarchy

`FtpCredentials` SHALL be a sealed hierarchy with `Password` and `PrivateKey`
variants. The `Password` variant SHALL require `username` (plain) and
`passwordSecretKey`. The `PrivateKey` variant SHALL require `username` (plain),
`privateKeySecretKey`, and OPTIONAL `passphraseSecretKey`. `PrivateKey`
credentials SHALL be rejected at validation time for non-SFTP protocols
(FTP, FTPS).

#### Scenario: SFTP with private-key credentials

- **GIVEN** a location with `protocol = sftp` and `credentials.type = privateKey`
- **WHEN** the configuration is validated
- **THEN** validation passes

#### Scenario: FTPS with private-key credentials (rejected)

- **GIVEN** a location with `protocol = ftps` and `credentials.type = privateKey`
- **WHEN** the configuration is validated
- **THEN** validation fails
- **AND** the error explains that private-key credentials require SFTP

### Requirement: Secret references use the SecretKey suffix

Any config field whose value is a lookup identifier for `SecretGetter` SHALL be
named with the `SecretKey` suffix (e.g. `passwordSecretKey`,
`privateKeySecretKey`, `passphraseSecretKey`, `authSecretKey`). The secret
value itself SHALL never appear in configuration. Usernames and other
operational identifiers SHALL be stored as plain config.

#### Scenario: All SecretGetter lookup fields use the SecretKey suffix

- **GIVEN** the configuration schema definition
- **WHEN** the schema is inspected
- **THEN** every field whose value is a `SecretGetter` lookup key ends in
  `SecretKey`
- **AND** no field named with the `SecretKey` suffix holds a raw secret value

### Requirement: Private key credentials are validated at startup

The watcher SHALL validate private-key credentials at startup. For every
location whose source is `LocationSource.Ftp` with `FtpCredentials.PrivateKey`,
the watcher SHALL resolve the secret via `SecretGetter`, parse the key with the
configured SSH library, and (if a passphrase is configured) decrypt the key,
all during startup. Failure to parse or decrypt SHALL fail startup with a clear
error naming the location and the failure reason.

> **Note:** this requirement is added to the spec in this change but is not
> exercised until `DirectTransport` lands (change 5). It is included here to
> establish the contract before any implementation needs to satisfy it.

#### Scenario: Valid private key parses successfully

- **GIVEN** a location with private-key credentials whose secret resolves to a
  valid key (and a valid passphrase, if applicable)
- **WHEN** the service starts
- **THEN** startup succeeds

#### Scenario: Invalid private key fails startup

- **GIVEN** a location with private-key credentials whose secret resolves to
  invalid key content
- **WHEN** the service starts
- **THEN** startup fails with a non-zero exit code
- **AND** the error names the offending location id and the parse failure

#### Scenario: Wrong passphrase fails startup

- **GIVEN** a location with private-key credentials whose passphrase secret is
  incorrect for the encrypted key
- **WHEN** the service starts
- **THEN** startup fails with a non-zero exit code
- **AND** the error names the offending location id and indicates a
  decryption failure (without revealing the passphrase)

### Requirement: Database state table exists and is migrated at startup

The watcher SHALL define the `ftp_file_state` table via a migration managed by
`db-toolkit`. Migrations SHALL run at startup, before the service accepts
traffic. Migration failure SHALL fail startup.

#### Scenario: Fresh database

- **GIVEN** a database with no prior schema
- **WHEN** the service starts
- **THEN** the migration runs successfully
- **AND** the `ftp_file_state` table exists with the documented schema
- **AND** `/health` returns 200 OK

#### Scenario: Database already at the latest migration

- **GIVEN** a database with all migrations already applied
- **WHEN** the service starts
- **THEN** the migration runner is idempotent (no-op)
- **AND** startup succeeds

#### Scenario: Migration fails

- **GIVEN** a database in an inconsistent state that causes the migration to fail
- **WHEN** the service starts
- **THEN** startup fails with a non-zero exit code
- **AND** the log message includes the migration error

### Requirement: ftp_file_state schema

The `ftp_file_state` table SHALL have the following columns and constraints:

- `location_id` TEXT NOT NULL
- `filename` TEXT NOT NULL
- `ftp_mtime` TIMESTAMPTZ NOT NULL
- `ftp_size` BIGINT NOT NULL
- `bundle_id` UUID NOT NULL
- `status` TEXT NOT NULL  (values: `DISCOVERED`, `DOWNLOADING`, `DOWNLOADED`, `PUBLISHED`, `FAILED`)
- `bucket_path` TEXT NULL  (populated when status reaches `DOWNLOADED`)
- `sha256` TEXT NULL  (populated when status reaches `DOWNLOADED`)
- `attempts` INT NOT NULL DEFAULT 0
- `last_error` TEXT NULL
- `discovered_at` TIMESTAMPTZ NOT NULL DEFAULT now()
- `updated_at` TIMESTAMPTZ NOT NULL DEFAULT now()
- PRIMARY KEY `(location_id, filename, ftp_mtime)`
- Partial index on `(location_id, status, ftp_mtime)` WHERE `status IN ('DISCOVERED', 'DOWNLOADING', 'DOWNLOADED')`

Nullable columns (`bucket_path`, `sha256`, `last_error`) are populated by later
state transitions. No business logic depends on these columns in this change;
the full schema is shipped to avoid follow-up migrations in every subsequent
change.

#### Scenario: Table inspection after migration

- **GIVEN** the migration has run on a fresh database
- **WHEN** the table is inspected
- **THEN** all columns, types, constraints, and indexes match the
  specification above

### Requirement: Health and status endpoints

The watcher SHALL expose two endpoints:

- `GET /health` — returns 200 OK once configuration has loaded and migrations
  have applied. This is the deployability signal used by Cloud Run, Kubernetes
  readiness probes, etc.
- `GET /status` — returns 200 OK with a JSON body describing operational state.
  In this change, the body reports the count of configured locations and a
  placeholder for per-location status to be populated by later changes.

#### Scenario: Health endpoint after successful startup

- **GIVEN** the service has started with valid configuration and migrations applied
- **WHEN** a client requests `GET /health`
- **THEN** the response is 200 OK

#### Scenario: Status endpoint with zero locations

- **GIVEN** the service is running with `locations: []`
- **WHEN** a client requests `GET /status`
- **THEN** the response is 200 OK
- **AND** the JSON body includes `{"locationsConfigured": 0, "locations": []}`

#### Scenario: Status endpoint with configured locations

- **GIVEN** the service is running with N configured locations
- **WHEN** a client requests `GET /status`
- **THEN** the response is 200 OK
- **AND** the JSON body includes `"locationsConfigured": N`
- **AND** the body includes an array entry per location with at least its `id`

### Requirement: Local development stack is reproducible

The repo SHALL include a Docker Compose file that brings up a local development
stack sufficient to run the watcher end-to-end against emulated dependencies,
and documentation explaining how to use it.

#### Scenario: docker compose up brings up the stack

- **GIVEN** a developer has Docker installed
- **WHEN** they run `docker compose up` from the repo root
- **THEN** Postgres, `fake-gcs-server`, and a Kafka broker (Redpanda) start
- **AND** the watcher container starts against them
- **AND** `GET /health` on the watcher container returns 200 OK

#### Scenario: docs/local-development.md exists

- **GIVEN** the repo
- **WHEN** a developer reads `docs/local-development.md`
- **THEN** the document explains:
  - How to start the dev stack
  - How to configure `platform.local.path` for unpublished platform-library
    changes
  - How to point the watcher at a real GCP project vs. the local emulator
  - Where logs and metrics surface locally
