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

Deleting an account must also delete its associated sender identities, contacts, trusted senders, account-wide preference overrides, and client-specific preferences.

This operation must **never delete the mailbox or any email messages on the mail server**.

#### RFDS005 — Account-level data isolation

Each account must have its own identities, contacts, trusted senders, and preferences.

All authorized sessions of the same account must have access to that account's persistent data. Sessions belonging to other accounts must not be able to access or modify it.

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

Examples include message-list page size, default message sorting, time-zone override, automatic marking as read, and preferred message content format. **Interface language and visual theme are client-specific preferences**, not account-wide settings.

#### RFDS020 — Three-level preference precedence

An account-wide preference's effective value must be resolved in this order, from highest to lowest priority:

1. The account's explicitly customized value.
2. The administrator's configured default.
3. Postela's built-in default.

The administrator must be able to customize the defaults without editing each account's preferences.

#### RFDS021 — Store only preference overrides

For **account-wide preferences**, Postela must retain only values explicitly customized by the user, rather than copying all defaults into each account.

When a user resets an account-wide preference, its customized value must be removed and the current effective default must apply. The distinct client-specific initialization rules are defined in RFDS026.

#### RFDS022 — Default changes and new preferences

When an administrator changes an **account-wide** default preference, accounts without a custom override must use the updated default.

New account-wide preferences added in later versions must be available to existing accounts without requiring them to create corresponding customized values.

The source and layering of application and administrator configuration are defined in [Core architecture](../technical/core-architecture.md).

### Client-specific preferences

#### RFDS025 — Client-specific preference isolation

Postela must retain client-specific preferences separately for each combination of **mail account and stable client application identifier (`client_id`)**.

The application identifier must not depend on a device, installation, or application version. A preference of one client application must not change or override the preference of another client application, even when both are used by the same mail account.

Client-specific preferences must not inherit backend or administrator TOML defaults. Their available values and meaning are defined by the client application.

#### RFDS026 — Persisting initial client language and theme

During successful login, a client must provide its `client_id` and may provide an initial `language` and/or `theme`.

For each supplied preference, Postela must save the initial value **only if that account and client application have no stored value for it**. An existing value must never be overwritten by an initial login value.

When no stored value exists and the client provides no initial value, that preference must remain unset. Unlike account-wide preferences, initial client-specific values may be persisted even before the user explicitly changes them.

#### RFDS027 — Retrieving and changing client-specific values

After the authenticated WebSocket connection is established, Postela must make the stored client-specific language and theme available to that client **only when a value exists**. It must not invent missing values.

A client must be able to change its stored language or theme by submitting its application identifier and selected value. The new value must be persisted for the authenticated account and that application without modifying preferences of other applications.

#### RFDS028 — Client-scoped preference notifications

When a client-specific preference changes, Postela must notify the other active sessions belonging to the **same mail account and client application**.

Sessions of the account using other application identifiers must not receive that client-specific update. Account-wide preference changes remain shared among all authorized sessions of the account, as described in RFDS023.

### Consistency across sessions

#### RFDS023 — Real-time propagation of account-data changes

When contacts, sender identities, trusted senders, or **account-wide preferences** are modified, Postela must notify the other active sessions authorized for the same account so their views can reflect the updated data.

Client-specific preference changes must follow the narrower account-and-client notification scope defined in RFDS028. No account-data change may be sent to sessions of unrelated accounts.

#### RFDS024 — Last write wins

If multiple sessions modify the same persistent account data, the **last modification processed by the backend must prevail**.

Postela must not require edit locks, conflict-resolution dialogs, or concurrent-edit version checks for this data.

After a successful modification, the resulting state must be made available to the account's other sessions.

## Scope boundaries

- **Authentication and sessions:** session tokens, passwords, credential handling, and session expiration are defined in [Core architecture](../technical/core-architecture.md) and are not persistent account data.
- **Mail messages:** Postela does not persist email messages; message reading and management will be specified in their own requirements group.
- **Mail-server configuration:** domain-level server configuration and per-account exceptions are administered by the backend rather than stored as user-customized mail-account records.
- **Client-specific behavior:** the language and theme login flow, supported values, and client-side presentation are specified in [General settings](./general-settings.md).
- **Technical storage design:** database selection, schema, transactions, access patterns, migration strategy, and integrity rules are defined in [Technical data storage](../technical/data-storage.md).
