# General settings

**Group:** `RFGS`

**Numbering:** `RFGS001`, `RFGS002`, `RFGS003`…

**Related functional requirements:** [Data storage](./data-storage.md) (`RFDS`).

## Scope

This group defines **user-facing account-wide settings** and related client display behavior.

Account-wide preference values are shared across all applications and sessions of the same mail account. Their effective defaults follow the precedence rules defined in [Functional data storage](./data-storage.md).

Client-defined configuration (for example interface language, visual themes, and layout preferences) is **not defined here**. Applications may persist arbitrary settings through the generic client-preference storage described in `RFDS` and its technical counterpart `RTDS`.

Administrative settings for mail servers, sessions, resource limits, backups, and security are outside this group. Postela does not require an administration interface.

## Functional requirements

### Account-wide preferences

#### RFGS001 — Shared account preferences and defaults

Preferences that affect backend mail operations must be associated with the mail account and shared by all authorized sessions, regardless of client application.

For account-wide settings with configurable defaults, the effective value must follow the agreed precedence: **user override, administrator TOML default, then Postela TOML default**.

Resetting a user override must restore the effective default without changing the administrator's configuration.

#### RFGS002 — Message-list page size

The user must be able to configure the number of messages shown per page as an account-wide preference.

Postela must provide an application default that the administrator can override, and the backend must enforce permitted minimum and maximum page sizes to prevent excessive requests.

#### RFGS003 — Default message sorting

The account must have a configurable default message sorting criterion and direction.

Postela's initial default must be **date descending (newest first)**. The administrator may change that default and the user may select a different sorting criterion and direction for their account.

#### RFGS004 — Temporary folder-list sorting

A client must be able to change the sorting of an open message list temporarily without modifying the account's default sorting preference.

The temporary sorting selection must apply to the list being viewed; it must not silently replace the user's saved default.

#### RFGS005 — Time zone preference

By default, clients must present dates and times using the time zone of the user's device.

The user must be able to select a different time zone as an account-wide preference. When such an override exists, it must be available to all sessions and clients for that account.

#### RFGS006 — Date and time presentation

Each client must be responsible for formatting dates and times for display, taking its selected language and applicable time zone into account.

The backend must exchange date and time values in a time-zone-independent manner, using UTC, rather than imposing a particular presentation format.

#### RFGS007 — Automatic marking as read

The user must be able to control whether opening an unread message automatically marks it as read.

This behavior must be **enabled by default**, subject to an administrator default override and the user's account-specific preference.

When enabled, opening an unread message must update its read state on the mail server and make that change visible to other sessions. When disabled, merely opening a message must not change its read state.

### Special folders

#### RFGS008 — Automatic identification of special folders

Postela must attempt to identify the account's special mail folders automatically, including:

- Sent
- Drafts
- Trash
- Junk
- Archive

These mappings must reflect the mail server's actual folders rather than assuming fixed folder names.

#### RFGS009 — Manual special-folder mappings

When automatic identification is missing or incorrect, the user must be able to select which existing folder serves each special-folder role.

The selections must persist per account and be shared across its sessions.

Changing a special-folder mapping must **not**, by itself, create, rename, or delete folders on the mail server.

### Message display

#### RFGS010 — Preferred message content format

The user must be able to choose whether messages should preferably be displayed as **HTML or plain text**.

This must be an account-wide preference with an application default of **HTML**, overridable by the administrator and by the user.

#### RFGS011 — Content-format fallback

When a message does not provide the user's preferred content format, Postela must allow the client to display the available format safely.

Choosing a preferred format must not authorize unsafe HTML handling or execution of active content.

#### RFGS012 — Remote-content protection remains applicable

HTML display must retain the default protection against loading external resources, subject only to the account's approved trusted-sender policy.

Selecting HTML as the preferred format must not automatically authorize remote resources or active content. The trusted-sender and remote-content behavior is specified in [Data storage](./data-storage.md).

## Scope boundaries and follow-up documentation

- **Client language and theme:** supported interface languages and visual themes belong to each client application. Their optional persistence uses the generic client-preference store defined in [Functional data storage](./data-storage.md) and [Technical data storage](../technical/data-storage.md).
- **Account-wide preferences:** defaults and overrides follow `RFDS020–RFDS022` and the layered TOML configuration requirements in [Core architecture](../technical/core-architecture.md).
- **Persistence:** this document specifies user-facing behavior, not database tables or schemas.
- **Mail operations:** special-folder detection, read-flag changes, and message content retrieval will be detailed in their own technical requirements.

