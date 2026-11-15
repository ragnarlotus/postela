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

#### RFRM003 — IMAP quota information

When the IMAP server supplies quota information, Postela must make available the usage and assigned limits for `STORAGE`, `MESSAGE`, and any other quota resources returned by the server.

It must preserve the association between mailboxes and their **quota roots**. Multiple mailboxes sharing a quota root must not cause its usage to be counted repeatedly or represented as separate storage allocations.

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

The mail server's permissions and restrictions must always be enforced, regardless of administrator policy. Postela must not assume that a single IMAP folder-deletion operation also deletes descendants.

#### RFRM011 — Folder-change synchronization

Creating, renaming, moving, or deleting folders must cause affected folder structures, status data, and special-folder availability to be refreshed in other active sessions of the account through the agreed synchronization mechanism.

Changes must not affect or be disclosed to sessions of another account.

### Message listing, filtering, and sorting

#### RFRM012 — Individual-message listings

A mailbox folder listing must show **individual messages**, not grouped conversation entries.

It must support pagination and the user's effective default sorting preference. Clients may request a temporary different sorting criterion and direction without changing that account-wide preference.

#### RFRM013 — Message list metadata

Each message in a folder listing must provide the information needed to display:

- Its UID, identifying the message within its IMAP folder.
- Subject.
- Sender display name and address, when present.
- Message date and received/internal date, when available.
- Message size in bytes.
- IMAP flags, including read, answered, flagged, and draft status where present.
- Whether it contains downloadable attachments.

**Recipients (`To` and `Cc`) and message-body previews are not required for list entries.**

#### RFRM014 — Metadata-only page retrieval

For an ordinary message listing, Postela must retrieve only the messages and metadata needed for the requested page and must **not download complete message bodies** merely to render the list.

The backend must not retrieve every message's full content or attachments to sort or paginate a large mailbox. The server-specific sorting and pagination strategies are defined in `RTRM`.

#### RFRM015 — Quick folder-list filters

Within a selected folder, clients must be able to request these temporary filters:

- All messages.
- Unread messages (without `\Seen`).
- Flagged messages (with `\Flagged`).

Unread and flagged filters must be combinable. These quick filters apply **only to the selected folder, excluding subfolders**.

Sorting and pagination must operate on the filtered results. Quick-filter changes must not modify saved account preferences.

#### RFRM016 — Availability of server-side sorting

Postela must apply a requested sorting criterion when supported by the mail server.

When the server cannot provide the required sorting capability, Postela must **not download an entire folder's metadata to sort it locally**. It must return the server-provided message order and inform the client that the requested sorting is unavailable.

Detailed `CONTEXT=SORT`, `UID SORT`, and cache behavior belongs to `RTRM`.

### Individual messages, conversations, and source content

#### RFRM017 — Read one message

A `read_message` operation must return **only the message selected** using its folder and UID, without searching for related conversation messages.

It must provide the relevant message headers (`From`, `To`, `Cc`, `Reply-To`, `Subject`, dates), available HTML and/or plain-text content, MIME/attachment metadata, embedded-resource references, and IMAP flags.

#### RFRM018 — Read an entire conversation separately

A distinct `read_thread` operation must retrieve the message selected by folder and UID **together with all discoverable related messages** in the authenticated account.

Conversation content must not be embedded or grouped into ordinary folder-list entries. Each application chooses whether to request a single message or a thread and how to present it.

#### RFRM019 — Cross-folder conversation discovery

Conversation relationships must be derived from message threading identifiers, including `Message-ID`, `In-Reply-To`, and `References`, **not merely a shared subject**.

A conversation may include received and sent messages, as well as messages in any other folder. It must not depend on the continued existence or location of an arbitrarily designated principal message.

Moving or permanently deleting one message must not prevent other discoverable related messages from forming the conversation.

#### RFRM020 — Conversation ordering and physical duplicates

`read_thread` must return the related messages **in chronological order, oldest first**.

It must include the originating folder and UID for every message. If multiple physically distinct messages share the same `Message-ID`, Postela must return all copies rather than silently deduplicating them.

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

#### RFRM032 — Synchronize flag and count changes

Flag changes and bulk read-state operations must be reflected in other sessions of the account, including affected unread counts and message state, through the existing IMAP synchronization architecture.

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

#### RFRM038 — Delete an entire conversation

A distinct `delete_thread` operation must act on **all discoverable physical messages in the selected conversation**, including received and sent messages in different folders and duplicate copies with the same `Message-ID`.

It must respect the current account's `delete_to_trash` policy, with messages already in `Trash` permanently deleted as defined for individual deletion.

Before execution, the backend must make available the affected message count and source folders so the client can present them to the user as appropriate. **The client, not Postela, decides whether to ask for confirmation.**

#### RFRM039 — Partial failures of cross-folder operations

When a multi-message or cross-folder operation is only partially completed, Postela must report which message operations succeeded and which failed when that information is available.

The backend must not imply that deleting a conversation across separate folders is atomic, and must not automatically repeat previously completed destructive actions in a way that risks unintended additional changes.

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

Each client application manages message selection, including selection across multiple listing pages, and any user confirmation before destructive operations.

Postela must not maintain per-session UI selection state. Batch operations on selected messages must receive explicit UID identifiers from the client, while folder-wide operations such as `empty_folder` must identify the target folder.

### Message search

#### RFRM044 — Search within a folder and its descendants

A search scoped to a folder must include messages in **that folder and all its descendant subfolders**, regardless of hierarchy depth.

Each result must identify the IMAP folder that contains the matching message.

#### RFRM045 — Search across the entire account

A global search must cover **all available folders** of the authenticated account, including `Trash` and `Junk` by default.

Postela must not implicitly exclude those folders or limit the global search to the inbox or subscribed folders.

#### RFRM046 — Search criteria

The client must be able to filter messages by sender, recipients, subject, message content, dates, and message state/flags.

The search must be performed against the server through IMAP rather than against a local persisted message index.

#### RFRM047 — Combined criteria and logical operators

A search must support multiple simultaneous criteria with both **AND** and **OR** semantics.

**AND is the default**, requiring all chosen criteria to match. **OR** allows messages matching at least one criterion. The client must specify the desired operator when it differs from the default.

#### RFRM048 — Filter by attachment presence

Searches must accept a boolean attachment-presence criterion, distinguishing messages **with** and **without** downloadable attachments.

Inline `cid:` images used exclusively for rendering message content must not count as downloadable attachments merely because they are separate MIME parts. Downloadable files, including those also suitable for preview, must count as attachments.

#### RFRM049 — Filter by message size

Searches must accept a size threshold in **bytes** and support finding messages **larger than** or **smaller than** that value.

The client may display or collect the value in user-friendly units, but the backend contract uses bytes.

#### RFRM050 — Paginated search results without local indexing

Searches must return paginated results, with the containing folder and UID identifying each result, including when results span multiple folders.

Postela must not persist emails, maintain a local full-text search index, or require downloading all message bodies for searching.

The efficient implementation of multi-folder searching, combining results, and pagination belongs to `RTRM`.

### Deferred features

#### RFRM051 — Copying messages between folders is deferred

Although IMAP supports copying messages to another folder, **message copying is outside the initial implementation scope**.

Postela must not require a public copy operation in its first release. The capability may be reconsidered if a concrete user need justifies duplicating server-side messages and their storage cost.

#### RFRM052 — Subscription-management commands are deferred

Postela must not require public `SUBSCRIBE` or `UNSUBSCRIBE` operations in its first release.

Folder subscription state remains readable as specified in RFRM005. Subscription editing may be reconsidered separately, without restricting access to unsubscribed folders.

## Boundaries and follow-up documentation

- **General settings:** the user-facing `delete_to_trash` account preference and the administrator-controlled folder-deletion policies must be reflected in the corresponding `RFGS` and `RTGS` documents. Those files are not changed by this document.
- **Technical reading and management (`RTRM`):** IMAP `MOVE`, `COPY` fallback when needed for moving, flag operations, MIME retrieval, full-header retrieval, attachment handling, `SEARCH`, `SORT`, `ESORT`, `CONTEXT=SORT`, and ordered-UID cache lifetime and regeneration will be specified there. The initial preference is to use server-side ordering, with per-session UID caching only where necessary.
- **Synchronization:** refresh and notification behavior follows the existing account monitoring and session requirements, not a separate per-client mailbox watcher.
- **Server authority:** available mailbox operations, permissions, message flags, quotas, and folder-expiration policies remain constrained by the IMAP server.
