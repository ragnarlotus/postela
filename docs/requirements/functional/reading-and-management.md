# Reading and management

**Group:** `RFRM`

**Numbering:** `RFRM001`, `RFRM002`, `RFRM003`…

**Related requirements:** [General settings](./general-settings.md) (`RFGS`), [Data storage](./data-storage.md) (`RFDS`), [Technical general settings](../technical/general-settings.md) (`RTGS`), and [Sync and notifications](./sync-and-notifications.md) (`RFSN`).

## Scope

This group defines the **mailbox operations and observable behavior** Postela must make available to authenticated client applications: folder discovery and management, listing and reading messages, conversations, flags, moving and deleting messages, message searches, quotas, original-message downloads, and attachments.

The **administrator** configures the Postela backend and its policies; the **user** accesses email through an application **client**. Postela exposes capabilities and results independently of a client's interface. Visual presentation, UI selection, and confirmation dialogs belong to each client.

Messages and attachments remain on the IMAP server. Postela must **not persist message bodies, attachments, or a local mail search index**. The precise IMAP commands, concurrency, pagination algorithms, and ordered-UID caching are defined separately in Technical Reading and Management (`RTRM`).

## Functional requirements

### Folder discovery, quota, and subscriptions

#### RFRM001 — Complete mailbox folder hierarchy

Postela must provide the client with the full IMAP mailbox folder hierarchy available to the authenticated account, including each folder's full server-provided name, position in the hierarchy, and parent-child relationships.

The backend must preserve the server's namespace and hierarchy separators, rather than assuming a particular folder naming convention.

#### RFRM002 — Folder status and special-folder roles

For each folder, Postela must provide the total message count, unread message count, and subscription state when the server supplies them.

It must also indicate the applicable special-folder role (`Sent`, `Drafts`, `Trash`, `Junk`, or `Archive`), resolved according to the user, administrator, and IMAP `SPECIAL-USE` policies established in `RTGS`.

Missing status information must not be represented as a fabricated zero or an invented role.

#### RFRM003 — Storage usage and limit

Postela must expose **only the storage space used and the storage limit**, when those values are provided by the IMAP server through the `STORAGE` quota resource:

- **Used space:** `STORAGE` usage, expressed in bytes.
- **Storage limit:** `STORAGE` limit, expressed in bytes.

The information must be retrieved **on demand**, starting with the quota information applicable to `INBOX`. The initial implementation does not require querying every folder or exposing other quota resources such as `MESSAGE`.

Postela must **not present an `INBOX` quota as the account-wide total unless the information provided by the server establishes that scope**. It must not infer a global total by summing independent or overlapping quota roots. Unavailable values must be distinguished from valid zero values as specified in RFRM004.

#### RFRM004 — Quota unavailable

If the server does not support IMAP quota queries, or does not provide a quota or limit for a particular resource, Postela must report that information as **unavailable**, not as zero.

The backend must not invent a global quota by adding unrelated or overlapping quotas.

#### RFRM005 — Subscription status without subscription management

Postela must expose each folder's IMAP subscription status, but its mailbox listing must not be restricted to subscribed folders.

Changing subscriptions using `SUBSCRIBE` or `UNSUBSCRIBE` is **deferred** from the initial release. A missing subscription must not hide an otherwise available folder from the backend's folder inventory.

### Folder management

#### RFRM006 — Create folders and subfolders

The user must be able to create an IMAP folder and create subfolders beneath existing folders, subject to the server's naming, hierarchy, and permission restrictions.

Creating a folder must not automatically assign it a special-folder role.

#### RFRM007 — Rename an existing folder

The user must be able to change a folder's name while preserving its intended location in the hierarchy, subject to restrictions imposed by the IMAP server, including special handling of `INBOX` where applicable.

The operation must retain its messages and subfolders as supported by the mail server.

#### RFRM008 — Move a folder within the hierarchy

Moving a folder to another hierarchy location must be available as a **distinct functional operation** from renaming it, even if both use the same underlying IMAP mechanism.

The move must preserve the folder's name and its descendant folder hierarchy, subject to the server's capabilities and restrictions.

#### RFRM009 — Keep special-folder mappings after Postela-controlled changes

When Postela successfully renames or moves a folder, it must preserve the meaning of affected special-folder assignments, including those referring to descendant folders whose full IMAP names have changed.

- For explicit **user overrides**, Postela must update the stored `special_folders` mapping to the new full IMAP name.
- For an explicit **administrator TOML mapping**, Postela must leave TOML unchanged and create a suitable user override referring to the new name.
- For an automatically detected **`SPECIAL-USE` mapping**, Postela must refresh discovery without introducing an unnecessary user override.

If a folder is renamed or removed externally, an unavailable explicit mapping must be retained and reported, following `RTGS`; Postela must not guess the new name.

#### RFRM010 — Delete folders under administrator policy

Postela must support deleting IMAP folders, subject to administrator-configurable policies that control:

- Whether folders containing messages may be deleted.
- Whether a folder and its descendant subfolders may be deleted recursively.
- Whether folders identified as special may be deleted.

The initial Postela defaults for all three permissions are **disabled**, allowing only the deletion of empty, non-special folders without subfolders.

**Deleting `INBOX` must always be prohibited**, regardless of administrator policy. Postela must also prohibit renaming or moving `INBOX` itself while allowing operations on its subfolders if the server permits them.

The mail server's permissions and restrictions must always be enforced. Postela must not assume that a single IMAP folder-deletion operation also deletes descendants.

#### RFRM011 — Folder-change synchronization

Creating, renaming, moving, or deleting folders must cause affected folder structures, status data, and special-folder availability to be refreshed in other active sessions of the account through the agreed synchronization mechanism.

Changes must not affect or be disclosed to sessions of another account.

### Message listing, filtering, and sorting

#### RFRM012 — Configurable individual or grouped folder listings

A folder listing must support two presentation modes, controlled by the account-wide boolean preference `thread_view`:

- **`true` (Postela default):** show one entry per conversation represented in the selected folder, if the server supports the required threading capability.
- **`false`:** show individual messages as separate entries.

The administrator may override the default and the user may override the administrator's setting for their account, following the established user → administrator → Postela precedence.

If conversation grouping is unavailable on the connected IMAP server, Postela must return individual-message entries while preserving the saved preference and informing the client of the unavailable feature.

Both modes must support pagination and the applicable default sorting preference. Clients may temporarily request a different sorting criterion and direction without changing the account-wide preference.

#### RFRM013 — Message list metadata and safe message identifiers

Each individual message entry, or the representative message for a grouped conversation, must provide the information needed to display:

- Its UID, identifying the message within its IMAP folder.
- Subject.
- Sender display name and address, when present.
- Message date and received/internal date, when available.
- Message size in bytes.
- IMAP flags, including read, answered, flagged, and draft status where present.
- Whether it contains downloadable attachments.

A single-folder list response must include the folder name and its **`UIDVALIDITY` once at response level**. Clients identify messages using the combination of folder, `UIDVALIDITY`, and UID; the folder and `UIDVALIDITY` need not be duplicated inside every list entry.

**Recipients (`To` and `Cc`) and message-body previews are not required for list entries.** Grouped entries must also supply the conversation counters defined in RFRM055.

#### RFRM014 — Metadata-efficient page retrieval

Postela must not download complete message bodies or attachments merely to render a folder listing.

For individual-message listings, it must retrieve detailed metadata only for the requested page. For grouped listings, it may obtain the server-provided conversation structure and UID/flag sets needed to determine grouping, filters, order, and page boundaries, but it must not fetch the full message content or full metadata of every message solely to render a page.

The server-specific sorting, threading, pagination, and temporary in-memory caching strategies belong to `RTRM`.

#### RFRM015 — Quick filters scoped to the selected folder

Within a selected folder, clients must be able to request these temporary quick filters:

- All messages.
- Unread messages (without `\Seen`).
- Flagged messages (with `\Flagged`).

Unread and flagged filters must be combinable. **These filters apply only to messages physically present in the selected folder, excluding subfolders, regardless of `thread_multifolder`.**

In a grouped list, a conversation matches if at least one of its messages in that folder satisfies the active filters. Filtering must not split the conversation or reduce its reported message counts to matching messages only. When filters are active, the conversation's representative must be **the most recent message in the selected folder that matches those filters**; with no filters, it is the most recent message in that folder.

Sorting and pagination must operate on the filtered results. Quick-filter changes must not modify saved account preferences. Failures while retrieving information from other folders must not prevent applying these folder-local filters.

#### RFRM016 — Sorting availability and safe fallbacks

Postela must apply a requested sorting criterion when supported by the server and applicable to the listing. If `sender_name` sorting is unavailable, it must prefer `sender_email` as the fallback, preserving the configured direction. For other unavailable criteria, it must prefer date descending, followed by an available criterion where possible.

If no configurable sorting criterion is available, Postela must **still return messages in the order provided by IMAP**, without claiming the result is sorted. It must not download an entire folder's message metadata solely to implement an otherwise unavailable sort.

The original user preference must be preserved when a temporary sorting fallback is necessary. The client must receive the effective available sorting options and default. Detailed `SORT`, `ESORT`, `CONTEXT=SORT`, and cache behavior belongs to `RTRM`.

### Individual messages, conversations, and source content

#### RFRM017 — Read one message

A `read_message` operation must return **only the selected message**, identified by folder, `UIDVALIDITY`, and UID, without searching for related conversation messages.

It must provide the relevant message headers (`From`, `To`, `Cc`, `Reply-To`, `Subject`, dates), available HTML and/or plain-text content, MIME/attachment metadata, embedded-resource references, and IMAP flags.

Opening a single message must respect the account's `mark_as_read` preference.

#### RFRM018 — Read an entire conversation separately

A distinct `read_thread` operation must return **all discoverable related messages across the authenticated account**, including sent and archived messages, regardless of which message of the conversation initiated the request or which folder is open.

Its result must be the same set of related physical messages when initiated from any member of the same discoverable conversation. It must not be restricted by `thread_multifolder`, which controls grouped-list counters and thread-wide actions rather than reading.

A client displaying grouped folders may request `read_thread` when a group is opened; an individual-message view may request `read_message`. Full conversation bodies must **never** be embedded in folder-list entries.

In the initial release, `read_thread` requires the server to advertise `THREAD=REFS`. Where this capability is unavailable, Postela must expose `read_message` instead and inform the client.

#### RFRM019 — Cross-folder conversation discovery without local indexing

Conversation relationships must be derived from explicit message threading identifiers, including `Message-ID`, `In-Reply-To`, and `References`, **never merely from a shared subject or sender**.

A conversation may include received and sent messages, as well as messages in any other available folder. It must not depend on the continued existence or location of an arbitrarily designated root message. Postela must preserve relationships it can establish even when intermediate messages have been deleted or moved.

Postela must locate related messages on demand using the IMAP server; it must not persist a local message or conversation index. It must return messages actually found, without claiming it can prove that no additional related messages exist.

#### RFRM020 — Conversation ordering, physical copies, and identifiers

`read_thread` must return the related messages **in chronological order, oldest first**, irrespective of the folder or message selected as the entry point.

Every returned physical message must include its source folder, `UIDVALIDITY`, and UID. If multiple physically distinct messages share the same `Message-ID`, Postela must return **all copies**, not silently deduplicate them.

The client may choose to present duplicate copies together visually.

#### RFRM021 — Mark-as-read behavior when opening a thread

When the account's `mark_as_read` preference is enabled, `read_thread` must automatically mark as read **only the explicitly selected message** that initiated the request.

The read flags of all other conversation messages must be preserved. When the preference is disabled, opening either a single message or a thread must not itself change read flags.

#### RFRM022 — Raw original message download

Postela must allow a client to retrieve an individual complete original message as **`.eml`**, identified by folder and UID.

The supplied data must preserve the original message representation, including all headers, MIME content, and attachments, without rewriting it. The client chooses the downloaded filename. Postela must not retain a persistent copy.

#### RFRM023 — Complete original headers

A distinct `read_headers` operation must return **all original headers** of a message, including routing, authentication, and signature fields when present, without filtering or rewriting them.

It must not download the message body or attachments and must not change the read state. The client may present or copy the returned header text.

### Attachments and embedded resources

#### RFRM024 — Attachment listing and metadata

The detailed message view must identify downloadable attachments and their original names, MIME types, and sizes when available.

Attachment listing must not require downloading each attachment's full content.

#### RFRM025 — Individual attachment download on demand

A client must be able to request and download an individual attachment, preserving its original filename where safe and providing its MIME type.

The backend must retrieve the relevant content **on demand** from IMAP. It must not store message attachments in its database or as persistent files.

File preview and presentation remain client responsibilities.

#### RFRM026 — Embedded CID resources

Resources embedded in the message and referenced from HTML through `cid:` must be available for automatic display without the user authorizing externally hosted resources.

The backend must retrieve only the required embedded MIME parts when needed and must not fetch unrelated downloadable attachments automatically.

#### RFRM027 — Remote resource restrictions

Externally hosted resources in HTML email, including remote images, must remain blocked by default and may be loaded only under the previously approved trusted-sender policy.

Permitting embedded `cid:` resources or selecting HTML as the preferred format must **not** permit unsafe active email content or bypass remote-content protections.

**Related requirements:** RFDS015–RFDS018; RFGS010–RFGS012.

### Message flags and bulk changes

#### RFRM028 — Set and clear read state

The user must be able to mark one or multiple messages as read or unread by setting or clearing the IMAP `\Seen` flag.

Explicit read/unread changes must be available regardless of the account's automatic `mark_as_read` preference.

#### RFRM029 — Set and clear flagged state

The user must be able to flag and unflag one or multiple messages using the standard IMAP `\Flagged` indicator.

The backend must respect the flags the IMAP server allows the user to modify.

#### RFRM030 — Preserve other message flags

Operations changing `\Seen` or `\Flagged` must not inadvertently change unrelated flags such as `\Answered` or `\Draft`.

Other available IMAP flags and keywords must be reflected in message state as provided by the server. The initial release does **not** require a general-purpose editor for arbitrary custom IMAP keywords.

#### RFRM031 — Mark an entire folder as read

Postela must offer an operation to mark **all currently unread messages in one selected folder** as read, without the client selecting individual UIDs.

The operation must not include subfolders or download message bodies. It must modify only read flags.

#### RFRM032 — Synchronize individual flags and conversation counters

Flag changes and bulk read-state operations must be reflected in other sessions of the account through the established IMAP synchronization architecture, including affected folder unread counts, individual message state, and the grouped conversation counters `thread.unread` and `thread.flagged`.

Changing flags must not by itself change which messages belong to a conversation.

### Moving, deleting, and classifying messages

#### RFRM033 — Move one or more messages

Postela must allow the user to move one or more messages from a source folder to a different existing IMAP folder that permits message storage.

The operation must preserve message content, attachments, and flags to the extent supported by the server. It must notify clients of changes to the relevant folder listings and counts.

The client supplies the source folder and selected UIDs for a batch; the backend does not track selections across list pages.

#### RFRM034 — No separate archive or unarchive operations

Archiving and unarchiving are uses of the **same generic move operation**, with `Archive`, `INBOX`, or another chosen mailbox as the destination.

The backend must not require separate archive/unarchive APIs. Client applications may offer dedicated UI actions that call the generic move operation.

#### RFRM035 — Account preference for deletion behavior

The account-wide boolean preference `delete_to_trash` must control deletion behavior:

- **`true` (Postela default):** move deleted messages to the special `Trash` mailbox.
- **`false`:** permanently delete the selected messages.

The administrator must be able to override the default through TOML, and the user may override it for their account. The preference does not impose any retention duration on mail-server folders.

**Follow-up:** `delete_to_trash` must also be added to the account preference catalog in `RFGS` and `RTGS`.

#### RFRM036 — Delete selected messages

A `delete_messages` operation must accept a source folder and a **list of client-selected UIDs** and apply the account's deletion policy only to those messages.

When deleting messages **already in `Trash`**, Postela must delete them permanently regardless of `delete_to_trash`.

If the configured policy requires moving to `Trash` and that mailbox is unavailable, the operation must fail without permanently deleting the messages.

#### RFRM037 — Empty a folder without message selection

An independent `empty_folder` operation must accept **only the target folder**, not an explicit list of selected UIDs, and act on all messages within that folder **without touching subfolders**.

When `delete_to_trash = true`, it must move the messages to `Trash`; when false, it must delete them permanently.

Emptying `Trash` must always delete its contents permanently, regardless of the preference. If the requested move-to-trash policy cannot be satisfied because `Trash` is unavailable, the operation must fail rather than permanently deleting messages.

The backend must not track which messages were visually selected in a client's paginated list.

#### RFRM038 — Delete messages belonging to a conversation

A `delete_thread` operation must support deleting the **identified physical messages belonging to a conversation**, including distinct physical copies sharing the same `Message-ID`.

Its scope must respect the account-wide preference `thread_multifolder`: **`false`** affects the selected folder only; **`true`** includes related messages across the account, including `Sent` and `Archive`. The source folder remains relevant when an action originates from a grouped folder listing.

From an **open conversation**, the client may supply the folder/`UIDVALIDITY`/UID identifiers already returned by `read_thread`; Postela must not repeat conversation discovery solely to perform that action. The backend must apply the effective scope and validate message identifiers before modifying them.

From a **grouped listing**, the client can supply the representative identifier; Postela must locate the related messages needed for the effective scope before acting. Complete-conversation deletion must **not be restricted to the detailed conversation view**.

The operation must respect `delete_to_trash`; messages already in `Trash` are permanently deleted. Postela must not require proof that every possible related message was found before deleting the identified messages. It must report known partial results or failures without claiming more messages were deleted than actually were.

The client decides whether and how to present a confirmation before deletion. The backend must not force a separate confirmation workflow or require the client to have fetched a full conversation beforehand.

#### RFRM039 — Partial failures of cross-folder and conversation operations

When a multi-message or cross-folder operation is only partially completed, Postela must report which message operations succeeded and which failed when known.

The backend must not imply that deleting or changing a conversation across folders is atomic, and must not automatically repeat previously completed destructive actions in a way that risks unintended additional changes.

Postela must distinguish unsuccessful IMAP queries from a successfully completed discovery that found no further related messages. Query errors must be reported, and available results may be returned rather than blocking the entire conversation view or subsequent authorized actions. No `complete` boolean claiming a definitive conversation boundary is required.

Affected folder listings and counts must be synchronized with actual server state.

#### RFRM040 — Mark messages as spam

Marking one or more messages as spam must **move** them to the configured special `Junk` mailbox.

If the destination is unavailable, Postela must reject the operation without modifying those messages.

#### RFRM041 — Mark messages as not spam

Marking messages as **not spam** must move the selected messages from `Junk` to `INBOX`.

If the required destination is unavailable, the move must not take place.

#### RFRM042 — Classification and retention boundaries

Postela must not directly train or administer a server-side antispam engine as part of marking mail as spam or not spam; those initial operations consist of IMAP message moves.

Deleting messages from `Junk` must follow the same `delete_to_trash` policy as deleting from other non-Trash folders.

Retention and automatic expunging of messages in `Trash` and `Junk` remain **exclusively the responsibility of the mail server**. Postela must not offer a user-configurable number of retention days that it cannot enforce.

#### RFRM043 — Client-managed selection and confirmation

Each client application manages individual-message selection, including selection across multiple listing pages, and any user confirmation before destructive operations.

Postela must not maintain per-session UI selection state. Batch operations on selected messages must receive explicit folder/`UIDVALIDITY`/UID identifiers from the client, while folder-wide operations such as `empty_folder` identify the target folder.

A grouped-list thread operation may instead receive its representative message identifier for server-side resolution. Operations from an open conversation may receive the explicit message identifiers already returned by `read_thread`, avoiding redundant discovery.

### Message search

#### RFRM044 — Search within a folder and its descendants

A search scoped to a folder must include messages in **that folder and all its descendant subfolders**, regardless of hierarchy depth.

Each result must identify the IMAP folder that contains the matching message.

#### RFRM045 — Search across the entire account

A global search must cover **all available folders** of the authenticated account, including `Trash` and `Junk` by default.

Postela must not implicitly exclude those folders or limit the global search to the inbox or subscribed folders.

#### RFRM046 — Search criteria and date types

The client must be able to filter messages by sender, recipients, subject, message content, dates, and message state/flags.

Date searches must distinguish **message/sent date** (`sent_date`) from **received/internal date** (`received_date`), with optional start and end dates for either criterion. An unspecified date type must mean `sent_date`. Date criteria are evaluated by calendar days as supported by IMAP.

The search must be performed against the server through IMAP rather than against a locally persisted message index.

#### RFRM047 — Combined criteria and logical operators

A search must support multiple simultaneous criteria with both **AND** and **OR** semantics.

**AND is the default**, requiring all chosen criteria to match. **OR** allows messages matching at least one criterion. The client must specify the desired operator when it differs from the default.

#### RFRM048 — Filter by attachment presence

Searches must accept a boolean attachment-presence criterion, distinguishing messages **with** and **without** downloadable attachments.

Inline `cid:` images used exclusively for rendering message content must not count as downloadable attachments merely because they are separate MIME parts. Downloadable files, including those also suitable for preview, must count as attachments.

#### RFRM049 — Filter by message size

Searches must accept a size threshold in **bytes** and support finding messages **larger than** or **smaller than** that value.

The client may display or collect the value in user-friendly units, but the backend contract uses bytes.

#### RFRM050 — Paginated, individually identified search results without local indexing

Advanced searches must return **individual matching messages**, even when `thread_view = true`, rather than grouped conversations. This is the **initial design**, to be reviewed when client implementations can be evaluated.

Searches must return globally paginated results, with each result identified by its containing folder, `UIDVALIDITY`, and UID, including when results span several folders. A multi-folder search must present one combined result list rather than a separate page per mailbox.

Postela must not persist emails, maintain a local full-text search index, or require downloading all message bodies for searching. The efficient implementation of multi-folder searching, combining results, and pagination belongs to `RTRM`.

### Deferred features

#### RFRM051 — Copying messages between folders is deferred

Although IMAP supports copying messages to another folder, **message copying is outside the initial implementation scope**.

Postela must not require a public copy operation in its first release. The capability may be reconsidered if a concrete user need justifies duplicating server-side messages and their storage cost.

#### RFRM052 — Subscription-management commands are deferred

Postela must not require public `SUBSCRIBE` or `UNSUBSCRIBE` operations in its first release.

Folder subscription state remains readable as specified in RFRM005. Subscription editing may be reconsidered separately, without restricting access to unsubscribed folders.


### Conversation-aware listings, preferences, and actions

#### RFRM053 — Grouped conversation view preference

The account-wide boolean `thread_view` must control whether folder listings contain conversation entries or individual messages. Its initial Postela default is **`true`**, with administrator and user overrides according to the established preference precedence.

This setting affects the folder-list presentation and the client's default choice between `read_thread` and `read_message`; it must not make `read_message` unavailable. A client must know when the server cannot support grouping, without losing the saved preference.

#### RFRM054 — Reliable grouping and per-folder presence

Grouped listings must identify conversation membership from explicit message relationships rather than grouping unrelated messages solely because their subjects match.

Postela must group conversations **within the selected folder**. A conversation appears in a folder listing if and only if at least one physical message belonging to it exists in that folder. The same conversation may therefore appear in multiple folder listings, but only once per folder.

The representative must belong to the listed folder. It is the newest message from that folder unless active quick filters require the newest matching message, as specified in RFRM015. Conversation data must be retrieved on demand without maintaining a persistent message index.

#### RFRM055 — Configurable multi-folder conversation scope

An account-wide boolean preference `thread_multifolder` must control the scope of **grouped conversation counters and actions**. Its initial Postela default is **`false`**, overridable by the administrator and user.

- **`false`:** count and act on related messages only in the selected folder.
- **`true`:** count and act on related messages across all available folders of the account, including `Sent`, `Archive`, and other folders.

This preference replaces the previously considered `thread_actions_scope` setting; `thread_actions_scope` must not be introduced as a separate preference.

`thread_multifolder` must **not** change which conversations appear in a particular folder listing, the scope of quick filters, or the account-wide scope of `read_thread`.

#### RFRM056 — Aggregated conversation counters and indicators

Each grouped conversation entry must include:

- `thread.count`: number of physical messages in the effective scope.
- `thread.unread`: number of those messages without `\Seen`.
- `thread.flagged`: number of those messages with `\Flagged`.

A grouped conversation is presented as unread if `thread.unread > 0` and flagged if `thread.flagged > 0`. A flagged indicator in a grouped list must be **informational**: toggling individual-message flags belongs to a message view, not a generic group-level star action.

**Folder counters remain strictly folder-local**, even when `thread_multifolder = true`. The same global conversation counters may appear in more than one folder's grouped list, but must not be added to folder counters or treated as additional physical messages.

If an external-folder query fails and a global conversation counter cannot be determined correctly, Postela must return **`null` for the affected counter**, with error information, rather than presenting a partial number as exact. The folder-local quick filters must remain usable.

#### RFRM057 — Quick filters on grouped entries

When `thread_view = true`, the quick filters from RFRM015 must select conversations according to whether **at least one message in the currently displayed folder** matches the requested read/flag conditions.

The whole conversation remains represented by one entry. A filtered entry's representative is the **newest matching message in that folder**. The conversation's counters retain their full effective scope and must not be reduced to only the matching messages.

The application must not search other folders merely to decide whether a grouped conversation satisfies a quick filter, even if `thread_multifolder = true`.

#### RFRM058 — Explicit read and unread actions on grouped conversations

From a grouped folder listing, the user must be able to mark a conversation as read or unread.

The operation must affect the messages belonging to that conversation in the scope defined by `thread_multifolder`:

- Mark as read: set `\Seen` on the applicable unread messages.
- Mark as unread: clear `\Seen` on all applicable messages.

These **explicit group-wide actions** are separate from automatically marking the selected message as read on `read_thread`; the latter must continue to affect only the message the user opened when `mark_as_read` is enabled.

Affected per-folder state and grouped counters must be synchronized with active sessions.

#### RFRM059 — Grouped-list actions without full message payloads

A grouped folder-list entry must expose its **representative** (folder/`UIDVALIDITY`/UID) and aggregate counters, **not the full set of identifiers of all messages in the conversation**.

The client must be able to request a supported thread-wide operation, including deletion, using this representative. Postela must resolve the relevant physical messages within the effective `thread_multifolder` scope before executing it.

An open conversation may instead supply explicit physical-message identifiers already obtained through `read_thread`. This must avoid redundant conversation discovery and must still respect the effective action scope, applicable account policies, and message identity validation.

#### RFRM060 — Group-aware sorting and pagination

A grouped folder listing must sort **conversations** and then paginate them. `page_size` must represent the number of conversation entries per page when `thread_view = true`, and the number of individual messages when grouping is disabled or unavailable.

The default representative and conversation ordering are based on the selected folder's messages, not the newest message elsewhere in the account. When quick filters are active, the newest matching message in that folder represents the conversation.

For `thread_multifolder = true`, Postela may calculate cross-folder counters **only for the conversations on the requested page**, after applying folder-local filters, sorting, and pagination. This optimization must not change the returned values or which conversations qualify for the page.

#### RFRM061 — Search result sorting and account-independent search defaults

Advanced search results must support temporary sorting by:

- `date` (message date).
- `folder`.
- `sender_name` (sender display name).
- `sender_email` (sender email address).
- `subject`.

The client must be able to request ascending or descending direction. In the absence of a request override, search defaults must come from administrator-overridable Postela TOML settings: **`[search].sort_by = "date"`** and **`[search].sort_direction = "desc"`** initially. These search defaults are separate from the saved account preference used for ordinary folder listings.

If `sender_name` is unavailable, use `sender_email` where supported while retaining the requested direction. Otherwise use an available date-descending sort, then another available criterion where possible. If the server offers **no usable sorting criterion**, Postela must still return the results in server-provided order, not refuse the search.

Sorting and pagination must apply to the combined multi-folder search result, not independently per folder. Effective sorting availability and defaults must be available to the client.

#### RFRM062 — Safe message identity for all message operations

Postela must provide and validate the combination **folder, `UIDVALIDITY`, and UID** whenever a client identifies a specific message.

This applies to reads, original headers and downloads, attachments, moves, deletions, spam classification, and explicit flag changes—not just destructive operations. A single-folder list may expose folder and `UIDVALIDITY` once at response level; each result spanning several folders must identify its own folder and `UIDVALIDITY`.

An operation must not act on a UID if the specified `UIDVALIDITY` no longer matches the selected server folder. It must report a specific identity-mismatch error rather than silently acting on a potentially different message.

#### RFRM063 — Quota retrieval and change notifications

Quota information must initially be retrieved **on demand** rather than requiring quota queries every time a client loads the folder tree.

After quota information has been requested for an account, Postela must refresh affected quota values when detected message and folder changes could affect usage, and notify relevant active sessions. Related events may be coalesced to avoid duplicate queries.

Clients must be able to distinguish unavailable quota information from a valid zero, and Postela must not calculate quota usage merely by guessing how much a message move or deletion changed storage.

The cache of quota information is transient and account-scoped; its lifecycle and server-specific refresh policy belong to `RTRM`.

#### RFRM064 — Capability-aware client behavior

Clients must be informed of effective Postela operations and options, including threading availability and supported sorting criteria, independently of raw IMAP capability names.

When `thread_view = true` but conversation grouping is unavailable, folder listings must remain usable as individual messages. When a sorting criterion is unavailable, the client must be told the effective supported options without making message retrieval unavailable.

The `session_capabilities` WebSocket protocol and its nested feature/options/default representation belong to `RTCA`, not to this functional document.

## Boundaries and follow-up documentation

- **General settings:** the account-wide `delete_to_trash`, `thread_view`, and `thread_multifolder` preferences, the search-sorting TOML defaults, and the administrator-controlled folder-deletion policies must be reflected in the corresponding `RFGS` and `RTGS` documents. Those files are not changed by this document.
- **Technical reading and management (`RTRM`):** IMAP `MOVE` (without a `COPY`/`STORE`/`EXPUNGE` move fallback), selective permanent deletion, MIME retrieval, header/attachment handling, `SEARCH`, `SORT`, `ESORT`, `CONTEXT=SORT`, `THREAD=REFS`, on-demand cross-folder relationship discovery, and per-session cache lifetimes and invalidation belong there. Postela must not create a persistent message or conversation index.
- **Synchronization:** refresh and notification behavior follows the existing account monitoring and session requirements, not a separate per-client mailbox watcher.
- **Server authority:** available mailbox operations, permissions, message flags, quotas, and folder-expiration policies remain constrained by the IMAP server.
