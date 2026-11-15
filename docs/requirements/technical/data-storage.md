# Data storage

**Group:** `RTDS`

**Numbering:** `RTDS001`, `RTDS002`, `RTDS003`…

**Related requirements:** [Functional data storage](../functional/data-storage.md) (`RFDS`) and [Core architecture](./core-architecture.md) (`RTCA`).

## Scope

This document defines how the Postela backend persists its own account data, protects database integrity, applies schema migrations, and performs scheduled and pre-migration backups.

It describes the **technical implementation** of behaviors established in `RFDS`. General backend configuration, session management, security, and service startup are defined in `RTCA`.

Postela must **never persist email messages, mail-server passwords, session tokens, or session state**. Email messages remain on the IMAP server. The database is reserved for Postela's own account records, sender identities, contacts, trusted senders, account-wide preference overrides, registered client applications, and client-specific preferences.

## Technical requirements

### Storage engine and database access

#### RTDS001 — SQLite storage engine

The backend must use **SQLite** as the initial persistent storage engine for Postela's application data.

The initial architecture must not require a separately managed database server, such as PostgreSQL or MariaDB.

**Related functional requirements:** RFDS001–RFDS034.

#### RTDS002 — Write-Ahead Logging mode

SQLite must operate in **Write-Ahead Logging (WAL)** mode to allow concurrent readers while a transaction writes data.

The implementation must respect SQLite's single-writer constraint. The database's `-wal` and `-shm` auxiliary files must be treated as part of its operational storage, not as independent backup files.

#### RTDS003 — Connection pool and busy handling

The backend must access SQLite through a **connection pool with a configurable maximum size**.

The application must allow concurrent reads and coordinate writes in accordance with SQLite's locking behavior. Database connections must use a configurable busy/wait timeout rather than immediately failing when a temporary write lock is encountered.

Connections and transactions must be released promptly after use.

#### RTDS004 — Foreign-key enforcement

Every SQLite connection must have foreign-key enforcement enabled using `PRAGMA foreign_keys = ON`, including connections used for migrations and other database maintenance.

Relationships between accounts and their associated records must be expressed through foreign keys.

#### RTDS005 — Atomic modifications and constraints

Operations that must update multiple records consistently must use database transactions.

The schema must apply `PRIMARY KEY`, `UNIQUE`, `NOT NULL`, and foreign-key constraints where appropriate. Data validation in Go complements, but does not replace, database integrity constraints.

Any multi-step operation must either commit a consistent result or roll back its uncommitted changes.

#### RTDS006 — Persistent-data isolation and cascades

All account-owned records must be associated with their owning account, and backend queries and modifications must be scoped to the authenticated account.

References to another account's records must not be accepted simply because their numeric identifiers exist.

Deleting an account must remove its owned sender identities, contacts, contact email addresses, contact telephone numbers, trusted senders, account-wide preferences, registered client applications, and their client-specific preferences using appropriate foreign keys and `ON DELETE CASCADE` rules.

Deleting data in Postela must not issue deletion commands against the remote mail server.

**Related functional requirements:** RFDS003–RFDS005.

### Account and identity schema

#### RTDS007 — Account identifiers and metadata

The `accounts` table must use a numeric internal `id` as its primary key and an `email` column with a uniqueness constraint as the account's application-level identifier.

It must record the account's creation timestamp and most recent successful-login timestamp.

Other account-owned tables must reference `accounts.id` through an `account_id` foreign key rather than duplicating the account email in every relationship.

**Related functional requirements:** RFDS001, RFDS002.

#### RTDS008 — Account registration transaction

On the first successful authentication, account registration must be performed atomically with creation of the account's first default sender identity.

On subsequent successful authentications, the existing account record must be reused and its last-login timestamp updated without resetting its original creation timestamp or duplicating its identity.

Failed authentications must not create persistent account records.

**Related functional requirements:** RFDS001, RFDS002, RFDS006.

#### RTDS009 — Sender identity schema

The `sender_identities` table must contain:

- `id`: internal primary key.
- `account_id`: owning account.
- `display_name`: sender display name.
- `reply_to`: optional reply address.
- `signature`: optional signature.
- `is_default`: whether the identity is the account's default.

It must **not contain a separately configurable sender/`From` email address**. The backend must derive and enforce `From` from the authenticated account when composing messages.

**Related functional requirements:** RFDS006–RFDS008.

#### RTDS010 — Exactly one default sender identity

The database must enforce **at most one default sender identity per account** using a partial unique index on the account identifier for identities marked as default.

Backend transactions must additionally maintain **at least one default identity per registered account**, including when identities are created, changed, or deleted.

Changing the default must atomically clear the previous default and set the new one. Deleting a default identity must not leave a registered account without a default identity.

**Related functional requirements:** RFDS006, RFDS007.

### Contacts schema

#### RTDS011 — Contacts table

The `contacts` table must contain the contact's internal `id`, owning `account_id`, name, and optional organization and notes.

A contact belongs to one account only. Contact management must not create a global address book or an independent recipient-history store.

**Related functional requirements:** RFDS009–RFDS011.

#### RTDS012 — Multiple contact email addresses

The `contact_emails` table must relate each email address to its parent contact through `contact_id`.

Each contact must support **one or more** email addresses without artificial fixed columns such as `email1` or `email2`. Deleting a contact must cascade to its email-address records.

Backend validation must prevent saving a contact without any email address.

**Related functional requirements:** RFDS010, RFDS011.

#### RTDS013 — Multiple contact telephone numbers

The `contact_phones` table must relate each optional telephone number to its parent contact through `contact_id`.

A contact may have multiple telephone numbers. Deleting a contact must cascade to its telephone-number records.

**Related functional requirements:** RFDS010, RFDS011.

#### RTDS014 — Contact import duplicate checks

During vCard import, the backend must check imported email addresses against existing contact email addresses **within the same account** before inserting contacts.

An imported address already associated with a saved contact must not result in a duplicate contact. The exact user-facing treatment of additional details on duplicate entries may be defined with the contact import workflow; it must not silently create a second contact for a matching address.

**Related functional requirements:** RFDS013, RFDS014.

### Trusted senders and preferences schema

#### RTDS015 — Trusted senders table

The `trusted_senders` table must contain:

- `id`: internal primary key.
- `account_id`: owning account.
- `email`: the authorized sender's exact email address.

The database must prevent duplicate addresses **within the same account**, using consistent email-address normalization/comparison that does not distinguish duplicates solely by letter case.

A trusted-sender record must apply to an individual address, **not to a domain wildcard**. It must not be interpreted as a guarantee that an incoming message genuinely originated from that address; the reading and security requirements will govern sender verification and safe remote-content loading.

**Related functional requirements:** RFDS015–RFDS018.

#### RTDS016 — Account-wide preferences table

The `account_preferences` table must contain:

- `account_id`: owning account.
- `key`: stable preference identifier.
- `value`: customized value represented as valid JSON.

The primary key must be the composite (`account_id`, `key`), preventing multiple overrides for the same preference in one account.

The backend must validate each account-wide preference's supported key, type, and value before saving it.

This table stores **only account-wide overrides**, not client-defined settings such as language, theme, or application layout. Generic client-specific settings are stored separately under RTDS032–RTDS033.

**Related functional requirements:** RFDS019–RFDS021.

#### RTDS017 — Preference overrides and resolution

Only **user-customized account-wide preference overrides** must be stored in `account_preferences`. Generic client-specific preference values have separate initialization and persistence rules under RTDS033 and RTDS037.

The effective value must be resolved in this precedence order:

1. Override in `account_preferences`.
2. Administrator default from `config.toml`.
3. Application default from `config.default.toml`.

Resetting a preference must delete its override row. New application defaults or administrator changes must take effect for accounts without corresponding overrides, without populating additional rows.

The TOML configuration merge and environment overrides remain governed by [RTCA004–RTCA008](./core-architecture.md).

**Related functional requirements:** RFDS019–RFDS022.

#### RTDS018 — Last-write-wins updates and notifications

Account data modifications, including both kinds of preference, must follow the **last write committed by the backend wins** policy.

The backend must not introduce edit locks, conflict-version checks, or conflict-resolution dialogs for this data.

After a successful transaction commits, the backend must notify authorized sessions through the existing WebSocket mechanism. **Account-wide changes** must reach the account's other active sessions; **client-specific preference changes** must reach only the same account-and-client-application sessions, as specified in RTDS039.

Notifications must not be emitted as successful updates before the underlying transaction has committed.

**Related functional requirements:** RFDS023, RFDS024, RFDS030.

### Schema versioning and migrations

#### RTDS019 — Versioned schema migrations

Every database schema change must be represented by an **ordered, versioned migration**, and the database must persist which migrations have been applied.

The backend must determine the current schema version and the pending migrations before performing application operations.

#### RTDS020 — Automatic startup migrations

On startup, the backend must check the database schema version and apply any required migrations **before accepting client connections**.

Migrations must execute in order and use transactions wherever technically possible. A migration must be recorded as applied only after its required changes have completed successfully.

#### RTDS021 — Mandatory pre-migration backup

**Before making any schema changes**, the backend must create a complete, consistent backup of the existing SQLite database.

Migration must not proceed unless the backup completes successfully. The backup must be identifiable as the state immediately preceding the attempted migration and available for recovery.

An unsuccessful pre-migration backup must be logged at `ERROR` level and must prevent startup.

#### RTDS022 — Migration failure and database recovery

If any migration fails, the backend must stop applying migrations, log an actionable `ERROR`, and **must not serve client requests using the failed or potentially incompatible schema**.

Changes made within a transaction must be rolled back when possible. The agreed recovery objective is to restore the **pre-migration database backup** and resume using the previous compatible Postela version.

The system must never attempt to start the updated executable against a restored schema that it requires to have been migrated.

#### RTDS023 — Application-version rollback dependency

A complete automatic rollback after a failed migration must restore **both**:

1. The pre-migration SQLite database.
2. The previous compatible Postela executable/release.

Automatic restoration and restart are permitted **only if the deployment/update mechanism can safely recover that previous release and coordinate the restoration**.

Until a compatible update and rollback mechanism is defined and implemented, a failed migration must leave Postela stopped with clear recovery information rather than claiming that an executable downgrade has occurred.

The release-retention and executable-rollback mechanism will be specified as part of the future update/deployment requirements, not invented within the database layer.

#### RTDS024 — Unsupported schema versions and downgrade policy

If the database schema is newer than the schema supported by the running Postela executable, the backend must refuse to start and report the incompatibility.

**Automatic reverse migrations are not required.** Recovery to the previous version is based on restoring the pre-migration backup together with a compatible application release, not on applying ad hoc reverse-schema transformations.

### Scheduled backups and retention

#### RTDS025 — Configurable automatic backup scheduling

Postela must support **scheduled automatic SQLite backups**.

Their frequency, backup directory, and maximum retained backup count must be configurable using the existing layered TOML configuration system (`config.default.toml` and optional `config.toml` overrides).

The backup scheduler must be managed by Postela itself, without requiring an external backup scheduler.

Manual backup commands, a dedicated CLI, and an administrative backup panel are **out of scope for the initial release**.

#### RTDS026 — WAL-safe consistent backups

Both scheduled and pre-migration backups must use **SQLite's backup API or an equivalently supported consistent SQLite backup mechanism** that correctly handles WAL mode and concurrent database activity.

Copying only the main `.sqlite` database file while WAL is active is not an acceptable backup procedure.

A backup must be considered valid only after the consistent backup operation has completed successfully.

#### RTDS027 — Backup retention and failure safety

Postela must enforce the configured maximum number of retained completed backups and remove older backups according to the retention policy.

**A failed or incomplete backup must never cause an existing valid backup to be deleted.**

The pre-migration backup needed for recovery from an active or failed migration must remain available until the migration/recovery outcome is resolved; routine retention must not remove the only required recovery point.

#### RTDS028 — Backup failure handling

A **scheduled backup failure** must be logged at `ERROR` level, but must not interrupt active sessions, mail operations, or the running Postela service.

A **pre-migration backup failure** must be logged at `ERROR` level, prevent the migration, and abort service startup.

A failed backup attempt must preserve previously valid backups.

#### RTDS029 — Restoration policy

Outside the conditional automatic migration-failure recovery described in RTDS022–RTDS023, Postela must **not automatically replace a running database with an older backup**.

The administrator must be able to restore a valid backup through a documented procedure performed while Postela is stopped.

The initial release does not require a restoration CLI or administrative user interface.

### Storage location and permissions

#### RTDS030 — Configurable persistent database location

The SQLite database file must be stored at a path configurable through TOML, inside or associated with a dedicated persistent-data directory **separate from the application code and executable**.

Deploying or updating Postela must not overwrite, reset, or delete this persistent-data directory.

#### RTDS031 — Database and backup file protection

The SQLite database, its `-wal` and `-shm` auxiliary files, and their containing data directory must have filesystem access limited to the operating-system identity running Postela, subject to the access needed for intended administration and recovery.

The backup directory must be independently configurable and protected against unauthorized access.

Data and backup paths must not expose persistent files as publicly served application resources.

### Registered applications and generic client preferences

#### RTDS032 — Registered client applications

The backend must maintain an `account_clients` table with:

- `account_id`: foreign key referencing `accounts.id`.
- `client_id`: stable client application identifier.
- `last_login_at`: timestamp of the most recent successful login by that account and application.

The composite primary key must be (`account_id`, `client_id`). Each registration belongs to one account, even when no preferences have yet been stored.

The table must support cascading deletion when the owning account is removed. The `client_id` is an application identifier, **not an authentication credential or separate authorization boundary**.

**Related functional requirements:** RFDS025–RFDS026, RFDS032–RFDS034.

#### RTDS033 — Generic client preferences table

The `client_preferences` table must contain:

- `account_id`: the owning mail account.
- `client_id`: the registered client application.
- `key`: arbitrary application-defined preference name.
- `value`: JSON value stored with its original JSON type.

The composite primary key must be (`account_id`, `client_id`, `key`), with a composite foreign key (`account_id`, `client_id`) referencing `account_clients` (`account_id`, `client_id`) and **`ON DELETE CASCADE`**.

No preference name or application-specific schema is reserved by Postela. Language, theme, and other frontend settings are ordinary examples, not specially handled fields.

The backend must never use account-wide preference fallback rules or TOML defaults to synthesize values for missing client preferences.

**Related functional requirements:** RFDS025, RFDS027–RFDS029.

#### RTDS034 — TOML-configured resource limits

The initial backend configuration must define these defaults, individually overridable through the layered TOML configuration system:

- Maximum **20** registered client applications per mail account.
- Maximum **200** stored preference keys per registered client application and account.
- Maximum **200** preference entries submitted in one operation.
- Maximum preference-key length of **128 Unicode characters**.
- Maximum serialized JSON value size of **65,536 bytes (64 KiB)**.
- Application inactivity retention of **30 days** since the most recent successful login.

These are configurable **limits**, not application-specific preference defaults. The backend must apply them to relevant login initialization, registration, and preference write operations.

**Related functional requirements:** RFDS031–RFDS034.

#### RTDS035 — Preference key and value validation

Preference keys must be nonempty valid UTF-8 strings within the configured maximum character length. Control characters must be rejected, but no mandatory naming convention or prefix may be imposed. Keys must be compared **case-sensitively**.

Values must be valid JSON and may be strings, numbers, booleans, `null`, arrays, or objects. The maximum size must be measured from the serialized JSON value in bytes. Reading and writing values must preserve JSON types without forcing all data into strings or converting arbitrary application-defined structures.

The backend must reject invalid entries with actionable errors, without modifying their existing values. It must not enforce application-specific meaning or a frontend preference schema.

**Related functional requirements:** RFDS025, RFDS029, RFDS031.

#### RTDS036 — Application registration and LRU rotation

Following successful authentication with a non-null `client_id`, Postela must reuse an existing `account_clients` registration when one exists. Otherwise, it must attempt to register the new account-and-application combination.

When the configured maximum number of registrations has been reached, the backend must select the registered client with the **oldest `last_login_at` that has no valid sessions**. Its registration and all associated preferences must be removed and replaced with the new registration.

The eviction and registration must be performed **atomically in one SQLite transaction**, with proper synchronization of concurrent logins so the maximum registration count cannot be exceeded.

Applications with valid sessions, including temporarily disconnected sessions still eligible for reconnection, must not be evicted. If no registration is eligible for removal, the new login must still succeed but **no client-specific preference storage may be initialized or accessed for that unregistered application** until a registration slot is available. Existing account-wide features remain available.

**Related functional requirements:** RFDS026, RFDS032.

#### RTDS037 — Login timestamp and insert-only preference initialization

For an existing or newly registered application, `last_login_at` must be set to the time of each **successfully completed login**. WebSocket traffic, authenticated operations after login, heartbeats, and reconnections of an existing session must not update it.

When the login includes a set of initial client preferences, each valid entry must be inserted **only if (`account_id`, `client_id`, `key`) has no stored value**. Use conflict-safe insert semantics so repeated or concurrent logins cannot overwrite saved preferences.

Entries that fail key, value, or quota validation must be reported individually; valid entries may still be stored. The overall login must not fail on preference errors. When the initial entry count exceeds the configured per-operation limit, reject the entire preference batch, **not the authenticated login**.

If the session has no `client_id` or application registration was unavailable, do not initialize client-specific preferences.

**Related functional requirements:** RFDS026–RFDS027, RFDS031–RFDS033.

#### RTDS038 — Reading and batch-writing preferences

When the authenticated WebSocket is established, the backend must make the existing preferences of the session's registered account-and-client application available without inventing absent values. It must also provide a single operation to read the complete set.

One write operation must support creating or updating multiple keys, preserving JSON value types. It must independently validate entries and provide per-key results for invalid entries while allowing valid changes to complete.

A write request exceeding the configured entry-count limit must be rejected in full. When a registered application has reached its stored-key quota, writes to existing keys may continue, while new keys must be rejected.

All reads and writes must be scoped to the authenticated account and its **session-bound client identifier**. A session with `client_id = null`, or an unregistered client identifier, must not access this storage. The public API must not expose deletion of individual client-preference keys.

**Related functional requirements:** RFDS026, RFDS028–RFDS031.

#### RTDS039 — Scoped change notifications

After a successful preference write has committed, the backend must send **one event for the operation's successful changes** to the other active WebSocket sessions matching both the authenticated `account_id` and `client_id`.

It must never notify other client applications, other accounts, or sessions without a client identifier of application-specific changes. Rejected entries must not be reported as successful changes.

Account-wide preference notifications are unaffected and continue to reach all authorized sessions of the owning account.

**Related functional requirements:** RFDS023, RFDS029–RFDS030.

#### RTDS040 — Daily inactivity cleanup

The backend must run a daily internal maintenance task to find `account_clients` records whose `last_login_at` is older than the configured inactivity period, initially **30 days**.

Before deleting a registration, the backend must verify that no **valid session** exists for that account-and-client application. An application with a valid session must be retained, even if its last recorded login exceeds the retention period.

Delete eligible registrations through their foreign-key cascade, including all `client_preferences`. A successful deletion must free the application registration slot. The task must not require an external scheduler, public API, administrative panel, or CLI.

**Related functional requirements:** RFDS032–RFDS034.

#### RTDS041 — Storage consistency and race safety

Account deletion, LRU replacement, preference initialization, quota enforcement, batch updates, and inactivity cleanup must preserve database integrity under concurrent sessions.

Insertion and quota checks must not allow more registered applications or preference keys than the configured maximum. Before rotating or cleaning a registration, the backend must account for valid sessions, not merely whether a WebSocket is currently connected.

Partial errors in a batch must not corrupt previously stored preferences. Notifications must describe only committed changes and must not be emitted before their database transactions complete.

**Related functional requirements:** RFDS024–RFDS034.

## Open implementation details

The following are intentionally not yet fixed:

- Concrete TOML configuration key names for storage, backups, and client-preference quotas/retention; the approved client-preference limit defaults are fixed above.
- SQLite database filename and on-disk directory defaults.
- SQLite driver/library selection, pool size, and busy-timeout defaults.
- Migration naming and metadata format, and the specific mechanism for coordinating versioned releases.
- Backup schedule syntax, retention count defaults, and backup filename format.
- How the future updater retains and restores the previous compatible executable and safely coordinates restart.

These implementation details must be decided without changing the approved functional behavior, data ownership rules, or migration/backup safety guarantees.
