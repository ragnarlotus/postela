# General settings

**Group:** `RFGS`

**Numbering:** `RFGS001`, `RFGS002`, `RFGS003`…

**Related functional requirements:** [Data storage](./data-storage.md) (`RFDS`).

## Scope

This group defines the **settings users can configure** and the resulting behavior of Postela and its clients.

Settings fall into two categories:

- **Account-wide preferences:** shared across all applications and sessions of the same mail account. Their effective defaults follow the account preference rules defined in `RFDS`.
- **Client-specific preferences:** associated with a mail account and a client application identifier. Their available choices and initial values belong to the client application, not to the backend's TOML configuration.

Administrative settings for mail servers, sessions, resource limits, backups, and security are outside this group. Postela does not require an administration interface.

## Functional requirements

### Client-specific preferences

#### RFGS001 — Stable client application identity

A client application must identify itself using a stable application identifier (`client_id`).

The identifier must distinguish applications, not individual devices, installations, or application versions. Sessions of the same mail account using the same application identifier must share that application's preferences.

#### RFGS002 — Language selection by client

Each client application must be responsible for the languages it offers and the translations it provides.

A user's selected interface language must be associated with their mail account and the client application identifier. Postela's backend must not require a fixed catalog of available interface languages or impose an English or Spanish default on all clients.

#### RFGS003 — Visual theme selection by client

Each client application must define its available visual themes and how they are applied.

A selected theme must be associated with the mail account and the client application identifier. Postela's backend must not impose `light`, `dark`, `system`, or any other theme identifiers or defaults on clients.

#### RFGS004 — Initial client preferences at login

During login, the client must provide its application identifier and may provide an initial `language` and/or `theme`.

For each preference independently:

- If no stored value exists for that account and application, Postela must save the supplied initial value, if any.
- If a value already exists, Postela must **preserve it without overwriting it** with the value supplied during login.
- If neither a stored value nor an initial value exists, Postela must leave that preference unset.

An initial value supplied by the client is not an administrator-defined or backend-defined default.

#### RFGS005 — Delivering client preferences on connection

When the authenticated client establishes its session WebSocket, Postela must provide the stored language and theme for that account and application **when those values exist**.

If either preference has no stored value, Postela must not invent or return a value for it.

#### RFGS006 — Changing client-specific preferences

A client must be able to update its selected language or theme by submitting its application identifier and the selected value.

Postela must persist the new value for the authenticated account and that application, without affecting the same account's preferences for other applications.

#### RFGS007 — Synchronizing client-specific preferences

When a client-specific preference changes, Postela must notify the other active sessions of the **same account and application**, so they can apply the updated value.

Sessions using a different application identifier must not have their client-specific preferences changed or receive those application-specific updates.

#### RFGS008 — Client responsibility for supported values

The client must interpret and apply its language and visual theme preferences, including which values it supports.

Postela's backend must treat theme identifiers as opaque values, checking only their permitted format and length, without maintaining a theme catalog. Language identifiers may represent values such as `en`, `es`, or `pt-BR`, but the backend must not maintain the set of translations offered by each client.

### Account-wide preferences

#### RFGS009 — Shared account preferences and defaults

Preferences that affect backend mail operations must be associated with the mail account and shared by all authorized sessions, regardless of client application.

For account-wide settings with configurable defaults, the effective value must follow the agreed precedence: **user override, administrator TOML default, then Postela TOML default**.

Resetting a user override must restore the effective default without changing the administrator's configuration.

#### RFGS010 — Message-list page size

The user must be able to configure the number of messages shown per page as an account-wide preference.

Postela must provide an application default that the administrator can override, and the backend must enforce permitted minimum and maximum page sizes to prevent excessive requests.

#### RFGS011 — Default message sorting

The account must have a configurable default message sorting criterion and direction.

Postela's initial default must be **date descending (newest first)**. The administrator may change that default and the user may select a different sorting criterion and direction for their account.

#### RFGS012 — Temporary folder-list sorting

A client must be able to change the sorting of an open message list temporarily without modifying the account's default sorting preference.

The temporary sorting selection must apply to the list being viewed; it must not silently replace the user's saved default.

#### RFGS013 — Time zone preference

By default, clients must present dates and times using the time zone of the user's device.

The user must be able to select a different time zone as an account-wide preference. When such an override exists, it must be available to all sessions and clients for that account.

#### RFGS014 — Date and time presentation

Each client must be responsible for formatting dates and times for display, taking its selected language and applicable time zone into account.

The backend must exchange date and time values in a time-zone-independent manner, using UTC, rather than imposing a particular presentation format.

#### RFGS015 — Automatic marking as read

The user must be able to control whether opening an unread message automatically marks it as read.

This behavior must be **enabled by default**, subject to an administrator default override and the user's account-specific preference.

When enabled, opening an unread message must update its read state on the mail server and make that change visible to other sessions. When disabled, merely opening a message must not change its read state.

### Special folders

#### RFGS016 — Automatic identification of special folders

Postela must attempt to identify the account's special mail folders automatically, including:

- Sent
- Drafts
- Trash
- Junk
- Archive

These mappings must reflect the mail server's actual folders rather than assuming fixed folder names.

#### RFGS017 — Manual special-folder mappings

When automatic identification is missing or incorrect, the user must be able to select which existing folder serves each special-folder role.

The selections must persist per account and be shared across its sessions.

Changing a special-folder mapping must **not**, by itself, create, rename, or delete folders on the mail server.

### Message display

#### RFGS018 — Preferred message content format

The user must be able to choose whether messages should preferably be displayed as **HTML or plain text**.

This must be an account-wide preference with an application default of **HTML**, overridable by the administrator and by the user.

#### RFGS019 — Content-format fallback

When a message does not provide the user's preferred content format, Postela must allow the client to display the available format safely.

Choosing a preferred format must not authorize unsafe HTML handling or execution of active content.

#### RFGS020 — Remote-content protection remains applicable

HTML display must retain the default protection against loading external resources, subject only to the account's approved trusted-sender policy.

Selecting HTML as the preferred format must not automatically authorize remote resources or active content. The trusted-sender and remote-content behavior is specified in [Data storage](./data-storage.md).

## Scope boundaries and follow-up documentation

- **Client language and theme:** these are client-specific preferences, not account-wide defaults supplied by the backend. Their login initialization and storage behavior are deliberate exceptions to the general preference-override model.
- **Account-wide preferences:** defaults and overrides follow `RFDS020–RFDS022` and the layered TOML configuration requirements in [Core architecture](../technical/core-architecture.md).
- **Persistence:** [RFDS](./data-storage.md) and the [technical data storage requirements](../technical/data-storage.md) must be updated to distinguish client-specific stored values from account-wide preference overrides. This document does not define database tables or schemas.
- **Mail operations:** special-folder detection, read-flag changes, and message content retrieval will be detailed in their own technical requirements.

