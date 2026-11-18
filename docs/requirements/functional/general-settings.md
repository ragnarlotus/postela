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

#### RFGS002 — Folder-list page size

The user must be able to configure the number of entries shown per page as an account-wide preference.

When conversation grouping is active, one entry represents one conversation; otherwise, one entry represents one message. The same `page_size` preference applies to both modes.

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

### Deletion and conversation preferences

#### RFGS013 — Deletion behavior

The account-wide boolean preference `delete_to_trash` must determine what happens when a user deletes messages:

- **`true` (Postela default):** move deleted messages into the configured `Trash` folder.
- **`false`:** permanently delete messages.

Deleting messages already in `Trash` must always be permanent. When a move to `Trash` is required but the destination is unavailable, Postela must reject the operation rather than permanently deleting the messages.

The administrator may override the default through TOML and the user may override it per account. No message-retention period is defined by this preference; retention remains the mail server's responsibility.

**Related requirements:** RFRM035–RFRM038.

#### RFGS014 — Conversation grouping preference

The account-wide boolean preference `thread_view` must control whether a folder listing shows **one entry per conversation** or **one entry per individual message**.

- **`true` (Postela default):** show grouped conversations if the server supports Postela's required conversation capability.
- **`false`:** show individual messages.

The administrator may override the default through TOML and the user may override it per account.

When grouping is unavailable, Postela must return individual messages and inform the client of that limitation **without discarding the user's saved preference**. The client may choose `read_thread` for a grouped conversation and `read_message` for an individual message; the backend must not prevent an individual read solely because grouping is enabled.

The grouping preference must not redefine the scope of quick folder-list filters or the set of messages that `read_thread` may retrieve.

**Related requirements:** RFRM012, RFRM018, RFRM053–RFRM054, RFRM060.

#### RFGS015 — Multi-folder conversation scope preference

The account-wide boolean preference `thread_multifolder` must control the scope of **grouped conversation counters and explicit actions on a conversation**:

- **`false` (Postela default):** consider only related messages physically present in the selected folder.
- **`true`:** consider related messages throughout the account, including `Sent`, `Archive`, and other folders.

The administrator may override the default through TOML and the user may override it per account. This preference replaces the earlier proposed `thread_actions_scope` name; no separate `thread_actions_scope` preference must be introduced.

`thread_multifolder` must **not** affect which conversations appear in a particular folder listing, the scope of the quick filters (always the selected folder), or `read_thread` (always the discoverable conversation across the account). Individual folder counters must remain folder-local.

**Related requirements:** RFRM055–RFRM059.

### Search result defaults

#### RFGS016 — Search sorting defaults and temporary overrides

Advanced message searches must use **date descending** as Postela's initial default sort order. The administrator may configure a different default search field and direction through TOML, independently of the account-wide default sorting preference for ordinary folder listings.

A client may specify the field and direction for an individual search without updating account-wide preferences. The available search sort fields are `date`, `folder`, `sender_name`, `sender_email`, and `subject`, subject to actual server capabilities. Unavailable criteria must be handled according to RFRM061; lack of sorting support must not prevent search results from being returned.

**Related requirements:** RFRM061.

## Scope boundaries and follow-up documentation

- **Client language and theme:** supported interface languages and visual themes belong to each client application. Their optional persistence uses the generic client-preference store defined in [Functional data storage](./data-storage.md) and [Technical data storage](../technical/data-storage.md).
- **Account-wide preferences:** defaults and overrides follow `RFDS020–RFDS022` and the layered TOML configuration requirements in [Core architecture](../technical/core-architecture.md).
- **Persistence:** this document specifies user-facing behavior, not database tables or schemas.
- **Mail operations:** special-folder detection, read-flag changes, conversation grouping, deletion, and message content retrieval are defined in [Reading and management](./reading-and-management.md) (`RFRM`) and their respective technical requirements. Administrative TOML keys and implementation rules belong to `RTGS` and `RTRM`.

