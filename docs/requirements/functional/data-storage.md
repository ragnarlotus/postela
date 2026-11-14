# Data storage

**Group:** `RFDS`

**Numbering:** `RFDS001`, `RFDS002`, `RFDS003`…

## Scope

This group defines **which account-related data Postela must retain and how users can manage it**. It covers account records, sender identities, contacts, trusted senders for remote content, user preferences, and changes shared across sessions.

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

Deleting an account must also delete its associated sender identities, contacts, trusted senders, and customized preferences.

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

### Account preferences

#### RFDS019 — Account-wide preferences

Each account must have its own persistent user preferences, shared across its authorized sessions and devices.

Preferences may include language, visual theme, message-list page size, and other configurable account-level options.

#### RFDS020 — Three-level preference precedence

A preference's effective value must be resolved in this order, from highest to lowest priority:

1. The account's explicitly customized value.
2. The administrator's configured default.
3. Postela's built-in default.

The administrator must be able to customize the defaults without editing each account's preferences.

#### RFDS021 — Store only preference overrides

Postela must retain only preferences explicitly customized by the user, rather than copying all defaults into each account.

When a user resets a preference, its customized value must be removed and the current effective default must apply.

#### RFDS022 — Default changes and new preferences

When an administrator changes a default preference, accounts without a custom override must use the updated default.

New preferences added in later versions must be available to existing accounts without requiring them to create corresponding customized values.

The source and layering of application and administrator configuration are defined in [Core architecture](../technical/core-architecture.md).

### Consistency across sessions

#### RFDS023 — Real-time propagation of account-data changes

When contacts, sender identities, trusted senders, or preferences are modified, Postela must notify the other active sessions authorized for the same account so their views can reflect the updated data.

Changes must not be sent to sessions of unrelated accounts.

#### RFDS024 — Last write wins

If multiple sessions modify the same persistent account data, the **last modification processed by the backend must prevail**.

Postela must not require edit locks, conflict-resolution dialogs, or concurrent-edit version checks for this data.

After a successful modification, the resulting state must be made available to the account's other sessions.

## Scope boundaries

- **Authentication and sessions:** session tokens, passwords, credential handling, and session expiration are defined in [Core architecture](../technical/core-architecture.md) and are not persistent account data.
- **Mail messages:** Postela does not persist email messages; message reading and management will be specified in their own requirements group.
- **Mail-server configuration:** domain-level server configuration and per-account exceptions are administered by the backend rather than stored as user-customized mail-account records.
- **Technical storage design:** database selection, schema, transactions, access patterns, migration strategy, and integrity rules will be defined in the technical Data storage requirements.
