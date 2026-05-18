# AGENTS.md — Instructions for AI Coding Assistants

This file governs how AI assistants (Claude Code, Cursor, etc.) work in this repo.
It supplements OpenSpec's built-in workflow with project-specific gates.

## Pre-implementation checklist

Before writing any code for an OpenSpec change, the AI assistant SHALL:

1. **Read `openspec/project.md`** to understand the tech stack, conventions, and
   platform-library catalogue.
2. **Resolve unresolved facts** (see below) if the current change needs them.
3. **Apply the platform-library gate** (see below) for any concern covered by a
   platform library — including concerns that may be covered by libraries not yet
   catalogued.
4. **Confirm in `design.md`** which libraries will be used and the exact pinned
   versions, with any deviations noted.

Only then proceed to implementation.

## Platform-library gate

The org maintains versioned Gradle libraries for cross-cutting concerns
(bootstrap, config, secrets, DB access, observability, event routing, etc.).
When a change touches such a concern, the AI assistant SHALL:

1. **Check `openspec/project.md`** for an existing entry covering the concern.
2. **If the library is catalogued:**
   - Check `gradle/libs.versions.toml` for the pinned version.
   - Read the library's published API/docs (or local override if
     `platform.local.path` is set) to understand idiomatic usage.
     Do not reproduce library patterns from memory or prior context — fetch them.
   - Use the library. Do not reimplement bootstrap, config loading, secret
     retrieval, migration runners, metric registries, event routing, or any
     concern the libraries cover.
3. **If the library is NOT catalogued, do not assume one does not exist.**
   The catalogue in `project.md` is partial; the org may have a library that
   isn't listed in this repo yet. The assistant SHALL ask the user, in a
   single grouped prompt:

   > "I need to handle [concern]. Is there an org-standard library for this?
   > If yes, what is it and where does it live? If no, I'll proceed with a
   > justified inline approach."

   If the user confirms a library exists, add it to `project.md` (propose the
   diff for confirmation), then use it. If no library exists, proceed inline
   with a clear note in `design.md`.
4. **If the library lacks a needed feature**, surface it to the user before
   working around it. Do not silently re-implement.

## Resolving unresolved project facts

If `openspec/project.md` contains `<TBD>` markers and the current change needs the
information they cover, the AI assistant SHALL:

1. Identify which unresolved facts are needed for *this* change. Do not ask about
   facts unrelated to the change in flight.
2. Ask the user for the values in a **single grouped prompt** — not one question
   per fact across multiple turns.
3. Propose the diff to `project.md` and ask for confirmation before committing.
4. Only then proceed.

## Test gate

Every OpenSpec change SHALL include:

- **Unit tests** covering new logic.
- **Integration tests** covering any cross-component or external-system
  interaction (database, Kafka, GCS, FTP, HTTP proxy, etc.).

Additionally, **every scenario added or modified in a capability spec SHALL be
covered by at least one automated test** — unit, integration, or both depending
on the scenario's surface. The mapping between scenarios and tests SHALL be
discoverable, by either:

- **Test naming that references the requirement/scenario it covers**
  (e.g. `Config_rejects_PrivateKey_credentials_with_FTPS`,
  `App_boot_with_zero_locations_reports_empty_status`), or
- **A coverage matrix in `design.md`** mapping each scenario to the test(s)
  that verify it.

Choose one approach per change and apply it consistently.

The change is not complete until both gates pass: tests exist for new logic
and external surfaces (the change-level gate), AND every scenario added or
modified is covered by at least one test (the per-scenario gate). CI SHALL
fail if either is unmet.

`tasks.md` SHALL end with a "Testing" section listing the unit and integration
tests added. If integration tests do not apply (rare — only when a change is
purely internal logic with no external surface), `tasks.md` SHALL state this
explicitly with justification.

Test infrastructure conventions:

- **Postgres:** Testcontainers.
- **GCS:** `fake-gcs-server` (Docker, also via Testcontainers).
- **Kafka:** Redpanda or single-broker Kafka via Testcontainers.
- **SFTP:** Apache MINA SSHD embedded in-process, or Testcontainers SSHD image.
- **HTTP services / proxies:** WireMock.

## House style

### Configuration

- Application code reads typed config objects loaded via `ApplicationConfig`
  (HOCON / Typesafe Config). Never read env vars or system properties directly.
- Endpoint overrides for emulators / local backends go via typed config fields
  (e.g. `hostOverride` on storage config), not via SDK-specific environment
  variables. Production code does not know that emulators exist.

### Secrets

- Loaded only via `SecretGetter`. Never inline, never logged, never read from
  config files as raw values.
- Config fields holding `SecretGetter` lookup keys use the `SecretKey` suffix:
  `passwordSecretKey`, `privateKeySecretKey`, `passphraseSecretKey`, etc.
- The actual secret value never appears in config. Operational identifiers
  (usernames, hostnames) are not secrets and are stored as plain config.
- The watcher does not transform secret values returned by `SecretGetter`.
  Secrets are stored in the format the consuming library expects; if a key
  format mismatch is observed, the fix is at the secret store, not in the watcher.

### Type-level invariants

- **Encapsulate co-required fields as nested optional types.** When two or more
  fields must always be present together (or absent together), express this as
  a nested type made optional as a unit — not as independent nullable fields
  coupled by validation logic. Put the invariant in the type system.
- **Model exclusive alternatives as sealed hierarchies.** When a config or
  domain entity has multiple variants (e.g. transport modes, credential types),
  use a Kotlin sealed interface/class rather than a flag-enum with parallel
  nullable fields.
- **Lift shared config to a registry.** When the same configuration block
  appears in multiple places (e.g. proxies referenced by many locations),
  define it once at the top level with a name and reference it by name.
  Do not duplicate.

### Operational config

- **Explicit, not inferred.** Network paths, proxies, endpoints, regions,
  and other deployment-dependent values are configured explicitly per location.
  The watcher does not infer them from related fields (e.g. protocol, region).
- **Infrastructure concerns belong to platform components, not service config.**
  If a platform component (e.g. `DomainEventRouter`, `SecretGetter`) owns a
  class of configuration (brokers, secret backends, etc.), the service does
  not duplicate that config in its own schema. The service holds only domain-
  specific config (e.g. topic names, event types) and uses the platform
  component for everything else.

### Validation

- **Fail loud at startup, never lazy.** Any property that can be validated at
  startup SHALL be validated at startup. Structural config errors, secret
  resolution failures, credential parse failures, and any other deterministic
  failure that an operator could discover at deploy time SHALL fail the service
  start with a clear, specific error message — not surface as a runtime error
  on first use.

### Errors

- Classify errors at the boundary (transient / permanent / auth) so retry
  logic can act on the category, not on the underlying exception type.

## Things to never do

- Reimplement a concern covered by a platform library.
- Resolve secrets via any path other than `SecretGetter` — no direct Secret
  Manager SDK calls, no env-var reads, no config-file values.
- Read env vars or system properties from application code.
- Reproduce a platform-library pattern from memory.
- Commit `<TBD>` placeholders that the current change depends on.
- Skip or stub integration tests for changes with external-system interaction.
- Add or modify a scenario in a capability spec without adding a test that
  covers it.
- Add emulator-awareness (e.g. `if (isLocal) ...` branches) to production code.
- Couple co-required fields as independent nullables instead of nesting them.
- Infer operational config from other fields when explicit declaration is feasible.
