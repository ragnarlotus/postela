# General settings

**Group:** `RTGS`

**Numbering:** `RTGS001`, `RTGS002`, `RTGS003`…

**Related requirements:** [Functional general settings](../functional/general-settings.md) (`RFGS`), [Functional data storage](../functional/data-storage.md) (`RFDS`), [Technical data storage](./data-storage.md) (`RTDS`), and [Core architecture](./core-architecture.md) (`RTCA`).

## Scope

This document defines the **technical configuration and resolution of account-wide settings**, including administrator-controlled TOML defaults and limits, user-owned account preference overrides, and the detection and resolution of special IMAP folders.

**Administrator** refers to the person who deploys and configures the Postela backend. **User** refers to the person accessing their mailbox through a client application. **Client** refers to that application. These roles must not be conflated.

The administrator maintains backend configuration in TOML; users may customize supported account-wide preferences through a client. Postela does not require an administrative CLI or web panel.

The SQLite schema, preference transactions, and persistence guarantees belong to `RTDS`. Client-specific key-value preferences belong to `RFDS` and `RTDS`, not to this group.

**Message sorting, IMAP pagination, conversation discovery, cache invalidation, and on-demand regeneration belong to Technical Reading and Management (`RTRM`)**. This document specifies associated administrator-controlled cache capacities and preferences, not the IMAP operation algorithms.

## Technical requirements

### Account-wide settings and configuration

#### RTGS001 — Stable account preference keys

The backend must define and validate these Postela-owned keys in `account_preferences`:

| Key | Stored value |
| --- | --- |
| `page_size` | Integer folder-list page size (messages or conversations, depending on mode) |
| `sort` | Object specifying folder-list sorting criterion and direction |
| `delete_to_trash` | Boolean controlling trash-first versus permanent deletion |
| `thread_view` | Boolean enabling grouped conversation listings when supported |
| `thread_multifolder` | Boolean controlling conversation counter and action scope |
| `timezone` | IANA time-zone identifier |
| `mark_as_read` | Boolean indicating whether opening a message marks it read |
| `message_format` | `html` or `text` |
| `special_folders` | Object mapping special-folder roles to full IMAP mailbox names |

These keys are **account-wide**: their effective values must be available to every authorized session of the account, regardless of its `client_id`.

The backend must not treat the arbitrary names used by `client_preferences` as Postela-owned account preference keys.

**Related requirements:** RFGS001–RFGS015; RFDS019–RFDS022.

#### RTGS002 — Administrator defaults and user overrides

For account-wide settings with built-in defaults, resolve effective values using this priority:

1. An explicitly stored user override in `account_preferences`.
2. The administrator's effective TOML configuration.
3. The corresponding Postela default in `config.default.toml`.

The administrator can override supported settings in `config.toml`, subject to the overall TOML and environment configuration precedence in [RTCA004–RTCA008](./core-architecture.md).

An account preference must not be copied into SQLite merely because a default exists. The **`timezone` and `special_folders` exceptions** are specified separately: neither requires a fixed Postela-provided value when the user has made no selection.

**Related requirements:** RFGS001; RFDS020–RFDS022; RTDS016–RTDS017.

#### RTGS003 — Validate account preference types and updates

The backend must reject unsupported account preference keys or values that do not match the defined type and allowed values.

Invalid updates must not alter the previously stored value. Updates to account-wide preferences must be scoped to the authenticated account and propagate to its other authorized sessions only after successful persistence.

No client may bypass the backend's account preference validation by supplying an arbitrary JSON value or a different `client_id`.

**Related requirements:** RFDS005, RFDS023–RFDS024; RTDS016–RTDS018.

### Folder-list page size and sorting preference

#### RTGS004 — Configurable page-size limits

The administrator must be able to configure these three values through layered TOML settings, with the following Postela defaults:

| Setting | Default |
| --- | ---: |
| Folder-list page size | **50** entries |
| Minimum allowed page size | **10** entries |
| Maximum allowed page size | **200** entries |

At startup, the backend must verify that **minimum ≤ default ≤ maximum**, and that the values are valid positive integers. An inconsistent configuration must fail validation under `RTCA007`.

A user's `page_size` override must be validated against the effective configured interval. A page contains **conversations** when grouping is active, and **individual messages** otherwise; the same preference and bounds apply to both.

**Related requirements:** RFGS002; RTCA004–RTCA008.

#### RTGS005 — Previously valid page-size override outside new limits

If an existing user's `page_size` override becomes invalid because the administrator changes the allowed interval, the backend must:

- Preserve the original value in `account_preferences` without silently rewriting or deleting it.
- Use the administrator's configured **default page size** as the effective value while the override is outside the interval.
- Inform the client that its saved preference cannot currently be applied.

The user may replace the override with a valid value. If the administrator later changes the interval so that the preserved value is valid again, that value must become effective automatically, provided the user has not replaced it.

**Related requirements:** RFGS002; RFDS020–RFDS022.

#### RTGS006 — Default sorting preference

The account-wide `sort` preference must store the requested **sorting criterion and direction** as one object.

The Postela default must be **date descending (newest first)**, and the administrator may replace this default through TOML. A user may store an account-specific preference.

The backend must validate the structure and the supported sorting options, with the actual available criteria, IMAP capability handling, and temporary per-list sorting defined by `RTRM`.

Changing a message list's sorting temporarily must not update the account-wide `sort` override unless the client explicitly requests that preference change.

**Related requirements:** RFGS003–RFGS004.

#### RTGS007 — Configurable ordered-UID cache capacity

The backend must expose a TOML-configurable limit for the number of UIDs retained by a message-sorting cache **per authenticated session**.

The Postela default must be **200,000 UIDs per session**. This setting limits cache capacity; it does not change the account's message-list page size.

The implementation of the cache itself—including `CONTEXT=SORT` and `UID SORT`, whether to retain an over-limit result, cache lifetime, invalidation, and regeneration—belongs exclusively to `RTRM`.

This capacity is **independent** of the separate conversation-structure cache limit defined in RTGS025.

### Time zones and message presentation

#### RTGS008 — IANA time-zone override

The optional account-wide `timezone` preference must be a valid **IANA time-zone identifier**, such as `Europe/Madrid` or `UTC`. The backend must reject an identifier that does not resolve to a recognized time zone.

**When the user has no time-zone override, the backend must not impose or return a default time zone.** Each client must use the time zone of its device.

Backend date and time values exchanged with clients must use UTC. Clients are responsible for rendering dates and times in their selected language and effective display time zone.

**Related requirements:** RFGS005–RFGS006.

#### RTGS009 — Automatic mark-as-read setting

The account-wide `mark_as_read` preference must be a boolean, with a Postela default of **`true`**. The administrator may change that default through TOML, and the user may override it for their account.

When enabled, opening an unread message must cause its read state to be updated on the IMAP server and reflected in other account sessions. When disabled, opening the message must not by itself alter its read state.

The IMAP flag operations and message-reading flow are defined in `RTRM`, not here.

**Related requirements:** RFGS007.

#### RTGS010 — Preferred content format

The account-wide `message_format` preference must accept only **`html`** or **`text`**, with a Postela default of **`html`**. The administrator may change the default in TOML; the user may override it for their account.

If a message lacks the preferred representation, the available representation may be used safely.

Selecting HTML must **not** bypass the trusted-sender policy, remote-resource blocking, or the prohibition on active/unsafe email content. MIME content retrieval, HTML sanitization, and display implementation belong to `RTRM` and the applicable security requirements.

**Related requirements:** RFGS010–RFGS012; RFDS015–RFDS018.

### IMAP special folders

#### RTGS011 — Special-folder roles and attribute-based discovery

Postela must recognize these account-wide special-folder roles and their IMAP `SPECIAL-USE` attributes (RFC 6154):

| Role | IMAP attribute |
| --- | --- |
| `sent` | `\Sent` |
| `drafts` | `\Drafts` |
| `trash` | `\Trash` |
| `junk` | `\Junk` |
| `archive` | `\Archive` |

Automatic discovery must use the attributes returned by the IMAP server, **not heuristics based on localized or conventional mailbox names**.

Postela must not have built-in mailbox names in `config.default.toml`. A role that cannot be resolved through any applicable source must remain unassigned.

**Related requirements:** RFGS008–RFGS009.

#### RTGS012 — Optional administrator mappings per mail domain

The administrator must be able to configure explicit special-folder mappings **per email domain** through the layered TOML configuration.

The administrator can supply actual full IMAP mailbox names for selected roles. Mappings are optional and may be incomplete; the absence of a mapping for a role must not prevent other roles from being resolved.

These are administrator-managed server defaults, not client-specific preferences or user overrides. The administrator's choice must not be inferred from the client application.

**Related requirements:** RFGS008–RFGS009; RTCA009.

#### RTGS013 — Single account-wide user override object

User-customized special-folder assignments must be stored in **one `account_preferences` entry**, with key `special_folders` and a JSON object as its value.

The object must contain only roles explicitly overridden by the user, using the role identifiers `sent`, `drafts`, `trash`, `junk`, and `archive` as property names.

Each value must be the **exact full IMAP mailbox name returned by `LIST`**, including its namespace and hierarchy separator when present. Postela must not construct, translate, or normalize mailbox paths into guessed names or filesystem Maildir paths.

The record belongs to the authenticated mail **account**, identified internally by `account_id`; it must not be copied into `client_preferences` or duplicated per `client_id`.

**Related requirements:** RFGS009; RFDS019–RFDS021; RTDS016.

#### RTGS014 — Per-role resolution priority

Resolve each special-folder role independently using this priority:

1. The user's explicit mapping in `account_preferences.special_folders`.
2. The administrator's explicit mapping for the mail domain in TOML.
3. Automatic discovery from IMAP `SPECIAL-USE`.

An absent mapping means "try the next source." **An explicit mapping that refers to an unavailable mailbox must not silently fall back** to a lower-priority source.

This precedence differs intentionally from a general preference with a Postela built-in value: special folders have **no fixed Postela mailbox-name defaults**.

**Related requirements:** RFGS008–RFGS009.

#### RTGS015 — Validate mapped mailbox availability

Before using an explicitly selected special folder, the backend must verify that the mailbox still exists in the server's IMAP mailbox inventory.

If an administrator or user mapping points to a deleted, renamed, or otherwise unavailable mailbox, the backend must:

- Retain the explicit setting without changing or deleting it.
- Report the mapping as **unavailable** to the client.
- Avoid creating, renaming, deleting, or silently substituting any mailbox.

An unavailable explicit mapping does not authorize fallback to another mailbox for that same role.

**Related requirements:** RFGS009.

#### RTGS016 — Ambiguous SPECIAL-USE discovery

If the server advertises **exactly one eligible mailbox** for a special-use role, Postela may resolve it automatically.

If multiple mailboxes advertise the same role, Postela must not select one arbitrarily. It must report the **ambiguous** role and provide its candidate mailbox names to the client so the user can choose an explicit mapping.

An existing valid user or administrator mapping retains its configured priority despite such ambiguity.

**Related requirements:** RFGS008–RFGS009.

#### RTGS017 — Distinct resolved special folders

The five special-folder roles must resolve to **distinct IMAP mailboxes**. The same mailbox must not be treated as two different special-folder roles.

When a proposed user change would produce duplicate effective mappings—including mappings inherited from TOML or `SPECIAL-USE`—the backend must reject the change and leave the existing settings unchanged.

Duplicate explicit administrator mappings are invalid configuration and must be detected through administrative configuration validation. Where automatic server discovery would produce conflicting role assignments, Postela must not silently accept duplicate resolved mappings.

#### RTGS018 — Partial updates to special-folder overrides

A user must be able to update a single property or a subset of properties within `special_folders` **without replacing unrelated user mappings**.

The backend must apply the change atomically in SQLite, preserving properties not mentioned in the update and validating the resulting effective mappings before committing.

Concurrent updates must not lose unrelated properties merely because both sessions read the same prior JSON object. The transaction must preserve the previously agreed **last committed write wins** behavior for conflicting changes.

**Related requirements:** RFDS023–RFDS024; RTDS005, RTDS018.

#### RTGS019 — Reset individual special-folder assignments

Resetting one role must remove **only that property** from the user's `special_folders` JSON object. Other user mappings must remain intact.

The reset role must again follow the administrator mapping when present, or IMAP `SPECIAL-USE` otherwise.

When the object has no remaining user-defined mappings, the backend must remove the entire `special_folders` preference row, rather than storing an unnecessary empty override.

The update or reset must be committed before notifying other sessions.

**Related requirements:** RFGS001, RFGS009; RFDS021.

#### RTGS020 — Refresh special-folder state from IMAP

At successful session establishment, Postela must obtain the account's current IMAP mailbox list and available `SPECIAL-USE` attributes.

After that, changes to the mailbox inventory must be detected using the already agreed shared IMAP `NOTIFY` monitoring and its `MailboxName` events. Following restoration of the monitoring connection, the backend must refresh mailbox information to recover changes missed during disconnection.

The backend must recompute effective special-folder mappings and their availability when relevant mailbox information changes. **Periodic polling solely for special-folder detection is not required.**

This requirement uses the existing monitoring architecture; it must not establish a separate monitoring connection for each client session.

**Related requirements:** RFGS008–RFGS009; RTCA023–RTCA025; [Sync and notifications](./sync-and-notifications.md).

#### RTGS021 — Special-folder updates across sessions

After a user changes or resets a special-folder override, Postela must notify **all other active sessions of the same mail account**, regardless of their client application identifier.

The backend must also notify those sessions when a change in the IMAP mailbox inventory affects a special folder's resolved mailbox, availability, or ambiguity.

Notifications must reflect committed preference changes or the current server-observed mailbox state. They must not leak mailbox or preference information to another account.

**Related requirements:** RFDS005, RFDS023–RFDS024; RTDS018.

### Deletion and conversation settings

#### RTGS022 — Account-wide deletion preference

The backend must recognize `account_preferences.delete_to_trash` as a boolean, with the Postela default **`true`**. Its effective value must follow RTGS002: explicit account override, administrator TOML default, then Postela default.

- `true`: deletions from folders other than the configured `Trash` mailbox use the existing Trash destination.
- `false`: deletions are permanent.
- Deletion **from `Trash` itself must always be permanent**, regardless of the preference.

When trash-first deletion is required but `Trash` cannot be resolved or used, the operation must fail rather than fall back to permanent deletion. The preference must not introduce a mail-retention policy or modify special-folder mappings.

Validating actual IMAP message operations and their atomicity limitations belongs to `RTRM`, not this configuration document.

**Related requirements:** RFGS013; RFRM035–RFRM039; RTGS011–RTGS015.

#### RTGS023 — Account-wide conversation view preference

The backend must recognize `account_preferences.thread_view` as a boolean, with the Postela default **`true`** and the effective priority defined in RTGS002.

When enabled and the mail server supports **`THREAD=REFS`**, folder listings use conversation entries; when disabled, they use individual-message entries. If the required capability is unavailable, Postela must return individual messages **without changing the stored `thread_view` override** and inform the client that grouping is unavailable.

This preference controls the list presentation and client's default choice between `read_thread` and `read_message`. It must not prohibit explicit single-message reads or turn advanced search results into grouped conversations.

Conversation discovery and capability handling belong to `RTRM` and `RTCA`.

**Related requirements:** RFGS014; RFRM012, RFRM018, RFRM050, RFRM053–RFRM054.

#### RTGS024 — Account-wide multi-folder conversation scope preference

The backend must recognize `account_preferences.thread_multifolder` as a boolean, with the Postela default **`false`** and the effective priority defined in RTGS002.

- `false`: grouped conversation counters and explicit thread-wide actions consider related physical messages **in the selected folder only**.
- `true`: they consider related physical messages **across all available account folders**.

This preference must not affect:

- Which conversations appear in a folder listing.
- The scope of quick filters, which always match messages physically present in the selected folder.
- The scope of `read_thread`, which always seeks the discoverable conversation across the account.
- Folder unread/message counts, which remain specific to each folder.

No separate `thread_actions_scope` preference must be introduced. When global counters cannot be determined reliably, returning `null` with error information rather than an apparently exact partial count is required by RFRM056; the error-handling strategy belongs to `RTRM`.

**Related requirements:** RFGS015; RFRM038, RFRM055–RFRM060.

#### RTGS025 — Configurable conversation-structure cache capacity

The administrator must be able to configure a separate limit for the number of UIDs retained **per session** by the `THREAD=REFS` conversation-structure cache.

The Postela default is **200,000 UIDs per session**. The limit must be a positive integer, validated as part of the effective backend configuration under RTCA007.

This limit is **independent** of the 200,000-UID default for the message-sorting cache in RTGS007. It must not reduce `page_size` or prevent a requested page from being calculated when the limit is exceeded: the backend may process the current request but must not retain an over-limit conversation structure in the session cache.

The `RTRM` document defines the IMAP construction, per-session/per-folder cache key, lifetime, invalidation on relevant folder changes or monitoring loss, reuse across WebSocket reconnections, concurrent regeneration, and cleanup.

**Related requirements:** RFRM054, RFRM060; RTGS007; RTCA007.

### Advanced search sorting defaults

#### RTGS026 — Administrator-configurable search sort defaults

Advanced searches must have **administrator-configurable sorting defaults independent of** `account_preferences.sort`. These defaults are backend TOML configuration rather than a user-owned account preference.

The Postela defaults in `config.default.toml` must be:

```toml
[search]
sort_by = "date"
sort_direction = "desc"
max_results = 500
```

`sort_by` must be validated against `date`, `folder`, `sender_name`, `sender_email`, and `subject`; `sort_direction` must accept `asc` or `desc`. Administrator overrides use the layered configuration rules of RTCA004–RTCA008.

A client may choose a different field and direction for an individual search request without writing an account preference or changing the administrator's defaults. The backend must announce effective available options and apply the agreed fallbacks when requested message metadata is unavailable. Global search-result sorting is performed by Postela using message metadata, without depending on server-side IMAP `SORT` support.

The metadata-based global ordering, multi-folder merge, and pagination algorithms belong to `RTRM`.

**Related requirements:** RFGS016; RFRM050, RFRM061; RTCA004–RTCA008.

### Administrator folder-deletion safeguards

#### RTGS027 — Configurable folder-deletion policies

The administrator must be able to configure three independent boolean controls, all **`false` by Postela default**:

| Setting | Meaning |
| --- | --- |
| `allow_delete_nonempty_folders` | Permit deletion of a folder that still contains messages, subject to the agreed empty-before-delete behavior. |
| `allow_recursive_folder_deletion` | Permit deletion of a folder together with its descendants. |
| `allow_delete_special_folders` | Permit deletion of folders resolved as special folders. |

Their effective values must follow the layered administrator TOML configuration and be validated as booleans. They are **administrator controls, not user-editable `account_preferences` keys**.

`INBOX` itself must **never** be renamed, moved, or deleted, regardless of these settings. Its subfolders remain eligible for operations where independently permitted. These controls may restrict operations further than IMAP, but must **never override the server's actual permissions**.

Folder deletion sequence, partial results, special-folder invalidation, and error behavior belong to `RTRM`.

**Related requirements:** RFRM007–RFRM011; RTCA004–RTCA008.

### Advanced-search processing limits

#### RTGS028 — Single administrator-configurable search limit

Provide exactly **one** administrator-only TOML setting, `[search].max_results`, initially **500** in `config.default.toml`. Require a positive integer, validate it with the layered configuration rules of `RTCA004–RTCA008`, and do not expose it as an account or client preference or allow a per-request override.

This **single configuration value** applies to both (a) the total number of messages matching an advanced search and (b) the number of candidate messages that must be inspected with `UID FETCH BODYSTRUCTURE` to evaluate attachment-presence conditions. No second limit for MIME inspection is defined.

If either count would exceed the configured value, reject the query with an identifiable limit-exceeded error requesting narrower search criteria; do not truncate matching messages, silently drop candidates, or display an arbitrarily limited subset as a complete result. The limit is **independent of folder-list `page_size` and advanced-search pagination**. Exact counting and early-rejection behavior belong to `RTRM014`.

**Related requirements:** RFGS017; RFRM048, RFRM050; RTRM014; RTCA004–RTCA008.

## Boundaries and deferred decisions

- **Administrator vs. user:** the administrator sets TOML defaults and safety limits; the user may save supported account-level overrides. These are different authorities and must remain distinct in naming, validation, and documentation.
- **Data storage:** table definitions, WAL, transaction semantics, default precedence storage, and account isolation remain in [RTDS](./data-storage.md). The generic per-client configuration system is out of scope.
- **Reading and management (`RTRM`):** IMAP message ordering and pagination, `CONTEXT=SORT`, `UID SORT`, `THREAD=REFS`, per-session sorting/thread cache lifetimes and invalidation, conversation discovery, and **regeneration on demand** belong there. This document specifies the independent cache capacities, not the cache algorithms.
- **Future implementation details:** TOML key paths other than the explicitly defined `[search]` section, API operation names/payloads, detailed error codes, and exact notification schemas are not fixed here. They must preserve the approved behavior and configuration-setting names above.
