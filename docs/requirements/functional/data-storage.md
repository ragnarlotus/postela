# Data storage

**Group:** `RFDS`

**Numbering:** `RFDS001`, `RFDS002`, `RFDS003`…

## Scope

This group defines **which account-related data Postela must retain and how users can manage it**. It covers account records, sender identities, contacts, trusted senders for remote content, account-wide preferences, client-specific preferences, and changes shared across sessions.

The requirements describe behavior, not database tables, storage engines, or implementation mechanisms. The technical data model, integrity constraints, and migrations will be defined separately.

**Postela must not store email messages locally.** Messages remain on the mail server. Authentication credentials, session tokens, and session state must not be persisted.

## Functional requirements

### Accounts and data ownership

#### RFDS001 — Automatic account registration

When a user successfully authenticates to Postela for the first time, Postela must automatically create a persistent record for that email account.

An unsuccessful authentication attempt must not create an account record.

#### RFDS002 — Account identification and access history

Each registered account must be uniquely identified by its email address.

Postela must retain the account's original registration date and the date of its most recent successful authentication.

#### RFDS003 — Account data independent of sessions

Persistent account data must remain available across logouts, session expiration, and subsequent logins.

An authentication failure or a period of inactivity must not delete a registered account or its data.

#### RFDS004 — Explicit account deletion

Removing a registered account from Postela must require an explicit action.

Deleting an account must also delete its associated sender identities, contacts, trusted senders, account-wide preference overrides, registered client applications, and their preferences.

This operation must **never delete the mailbox or any email messages on the mail server**.

#### RFDS005 — Account-level data isolation

Each account must have its own identities, contacts, trusted senders, and preferences.

All authorized sessions of the same account must have access to its account-wide persistent data. Client-specific preferences must be accessible only through sessions associated with the corresponding application identifier. Sessions belonging to other accounts must not be able to access or modify any of that account's data.

A shared, global contacts address book is outside the initial scope.

### Sender identities

#### RFDS006 — Default sender identity

Postela must automatically create an initial sender identity when an account is first registered and set it as that account's default identity.

#### RFDS007 — Multiple configurable identities

An account must support multiple sender identities, with one selected as the default.

Each identity must support a sender display name, an optional `Reply-To` address, and an optional signature. Users must be able to manage their identities and choose which one is the default.

#### RFDS008 — Fixed sender email address

**Every sender identity must use the authenticated account's email address as its `From` address.**

A user must not be able to configure a different `From` address through an identity. Postela must enforce this restriction regardless of values supplied by the client.

Identities may vary the display name, `Reply-To`, and signature, but not the sender email address.

### Contacts

#### RFDS009 — One address book per account

Each account must have a single, independently managed contacts address book.

Postela must not maintain a separate address book or recipient-history list for automatically remembered addresses.

#### RFDS010 — Contact information

A contact must support a name and one or more email addresses.

A contact may also include telephone numbers, an organization, and notes.

#### RFDS011 — Explicit contact management

Users must be able to create, edit, and delete contacts within their own account.

Contacts must only be added through an explicit user action. Postela must **not automatically create contacts from recipients of sent messages**.

#### RFDS012 — Contact-based address autocomplete

When entering email recipients, the client must be able to suggest addresses from the account's saved contacts.

Autocomplete must not depend on a separate automatically collected recipient history.

#### RFDS013 — vCard contact import

Users must be able to import contacts from a vCard (`.vcf`) file into their account's address book.

Import must avoid creating duplicate contacts when an imported email address already matches one saved in the account.

#### RFDS014 — vCard contact export

Users must be able to export their account's contacts to a vCard (`.vcf`) file.

Import and export must operate only on the authenticated account's address book.

### Trusted senders and remote content

#### RFDS015 — Remote-content blocking by default

When displaying HTML email, the client must **block loading externally hosted resources by default**, including remote images.

This default must apply even when the message includes references to external resources.

#### RFDS016 — Trusted-sender list

Each account must have its own persistent list of email addresses authorized for remote-content loading.

Users must be able to add and remove addresses from this list.

#### RFDS017 — Exact-address trust matching

A trusted-sender authorization must apply to an **exact email address**, not to an entire domain.

Authorizing one address must not automatically authorize other addresses at the same domain.

A sender address appearing in the list must not, by itself, be treated as proof of the message's authenticity. Verification and safe handling will be addressed in the technical reading and security requirements.

#### RFDS018 — Remote content from trusted senders

When the sender of an HTML message meets the applicable trusted-sender policy for the account, the client must allow remote resources to load without requiring the user to enable them for that message.

Trusting a sender must **never permit execution of active or unsafe email content**, such as JavaScript.

### Account-wide preferences

#### RFDS019 — Account-wide preferences

Each account must have its own persistent account-wide preferences, shared across all its authorized sessions and client applications.

Examples include message-list page size, default message sorting, time-zone override, automatic marking as read, and preferred message content format. Client-defined interface settings, such as language and theme, may instead be stored through the generic client-preference facility; they are not predefined account-wide settings.

#### RFDS020 — Three-level preference precedence

An account-wide preference's effective value must be resolved in this order, from highest to lowest priority:

1. The account's explicitly customized value.
2. The administrator's configured default.
3. Postela's built-in default.

The administrator must be able to customize the defaults without editing each account's preferences.

#### RFDS021 — Store only preference overrides

For **account-wide preferences**, Postela must retain only values explicitly customized by the user, rather than copying all defaults into each account.

When a user resets an account-wide preference, its customized value must be removed and the current effective default must apply. The distinct client-specific initialization rules are defined in RFDS027.

#### RFDS022 — Default changes and new preferences

When an administrator changes an **account-wide** default preference, accounts without a custom override must use the updated default.

New account-wide preferences added in later versions must be available to existing accounts without requiring them to create corresponding customized values.

The source and layering of application and administrator configuration are defined in [Core architecture](../technical/core-architecture.md).

### Consistency across sessions

#### RFDS023 — Real-time propagation of account-data changes

When contacts, sender identities, trusted senders, or **account-wide preferences** are modified, Postela must notify the other active sessions authorized for the same account so their views can reflect the updated data.

Client-specific preference changes must follow the narrower account-and-client notification scope defined in RFDS030. No account-data change may be sent to sessions of unrelated accounts.

#### RFDS024 — Last write wins

If multiple sessions modify the same persistent account data, the **last modification processed by the backend must prevail**.

Postela must not require edit locks, conflict-resolution dialogs, or concurrent-edit version checks for this data.

After a successful modification, the resulting state must be made available to the appropriate authorized sessions, respecting application-specific data isolation.

### Client-specific preference storage

#### RFDS025 — Generic client preferences

Postela must offer a persistent, generic **key-value store** for each combination of mail account and client application identifier (`client_id`).

Client applications must define their own preference keys, supported values, and interpretation. Postela must not predefine preference names or schemas, including for language and theme.

Values may contain any valid JSON type: string, number, boolean, `null`, array, or object. The backend must preserve JSON value types without interpreting their application-specific meaning.

These stored values must **not** fall back to administrator or Postela TOML defaults; the TOML configuration only governs the storage service and its limits.

#### RFDS026 — Optional client identification

A client may authenticate with or without a `client_id`. A session without an identifier must retain access to the account's general mail functions and account-wide preferences but must not use client-specific preference storage or receive client-specific preference notifications.

Each supplied `client_id` must consistently identify an application across versions, devices, and installations. Preferences belonging to other applications or accounts must remain isolated.

#### RFDS027 — Optional initial preferences at login

A client with a `client_id` may submit a batch of initial key-value preferences during login.

For each valid supplied key, Postela must persist the initial value **only when no stored value already exists** for that account and application. Existing values must never be overwritten by login defaults.

Invalid initial preferences must be rejected individually and reported with their reasons. A valid authentication must **not fail because of invalid initial preferences**; valid entries may still be stored.

A client that omits initial preferences must not have artificial preference values created on its behalf.

#### RFDS028 — Retrieving client preferences

Once the authenticated WebSocket is established, Postela must make all stored preferences of the session's account and registered application available together.

The client must also be able to request its complete set of stored preferences through one operation. An empty store must be returned as empty, without inventing values or applying account-wide defaults.

#### RFDS029 — Batch updates and partial errors

An identified client must be able to **create or update multiple preferences in one operation**. The backend must validate each entry independently, persist valid entries, and report rejected entries with the reason for each error.

A request exceeding the configured per-operation entry limit must be rejected in full, without applying any changes.

The initial public client-preference API must **not provide an operation to delete individual preference keys**. Internal deletion during account removal, maintenance, or application rotation remains permitted.

#### RFDS030 — Client-scoped change notifications

After a successful client-preference update, the backend must notify other active sessions of the **same account and `client_id`**.

A batch of changes must produce a **single notification event** for that operation, rather than a separate notification for every changed key. Sessions of other accounts or other client applications must not receive it.

#### RFDS031 — Configurable preference storage limits

The backend must enforce these initial limits, with values configurable through layered TOML settings:

- **200** stored preferences per account and client application.
- **20** registered client applications per account.
- **200** preference entries in one operation.
- **128** Unicode characters per preference key, with keys required to be nonempty and to contain no control characters.
- **64 KiB (65,536 bytes)** of serialized JSON per preference value.

Preference keys must be compared case-sensitively. Client-specific values must be validated as JSON but not against an application-defined schema.

Reaching the per-application key limit must prevent creating additional keys, **not** updating existing keys. Rejected writes must leave previously stored values unchanged.

#### RFDS032 — Rotation of registered applications

When a successful login introduces a new `client_id` and the account has reached its configured application limit, Postela must **replace the least recently logged-in eligible application**, deleting its registration and preferences before registering the new application.

An application with any **valid session** must not be selected for rotation. When all existing registered applications are protected by valid sessions, the new client must still be allowed to log in, but its application-specific preference storage must remain unavailable until capacity is available.

A login by an already registered application must not cause an application rotation.

#### RFDS033 — Tracking registered application logins

Postela must record the timestamp of the most recent successful login for each registered account and `client_id`.

This timestamp must be updated **only when a login successfully completes**. WebSocket activity, other authenticated operations, and connection maintenance must not update it.

The last-login timestamp must be used to determine application rotation order and maintenance inactivity.

#### RFDS034 — Automatic removal of inactive applications

Postela must run a daily maintenance routine to delete registered client applications and their stored preferences after a configured period with no successful login.

The initial inactivity period must be **30 days**, adjustable through TOML. Applications with any **valid session** must not be removed, even if their recorded last login is older than the configured period.

Deleting an inactive application must free its registration slot. No user-facing administration panel or CLI is required for this cleanup.

## Scope boundaries

- **Authentication and sessions:** session tokens, passwords, credential handling, and session expiration are defined in [Core architecture](../technical/core-architecture.md) and are not persistent account data.
- **Mail messages:** Postela does not persist email messages; message reading and management will be specified in their own requirements group.
- **Mail-server configuration:** domain-level server configuration and per-account exceptions are administered by the backend rather than stored as user-customized mail-account records.
- **Client applications:** the optional client identifier is bound to a session as specified in [Core architecture](../technical/core-architecture.md). The backend does not define client language or theme choices.
- **General settings:** user-facing account-wide preferences and message display behavior are specified in [General settings](./general-settings.md).
- **Technical storage design:** database selection, schema, transactions, access patterns, migration strategy, and integrity rules are defined in [Technical data storage](../technical/data-storage.md).
