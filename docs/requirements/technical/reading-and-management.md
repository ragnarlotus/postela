# Reading and management

**Group:** `RTRM`

**Numbering:** `RTRM001`, `RTRM002`, `RTRM003`…

**Related requirements:** [Functional reading and management](../functional/reading-and-management.md) (`RFRM`), [Technical general settings](./general-settings.md) (`RTGS`), [Core architecture](./core-architecture.md) (`RTCA`), and [Sync and notifications](./sync-and-notifications.md) (`RTSN`).

## Scope

This document specifies the technical implementation of mailbox reading and management through IMAP, following the functional requirements in `RFRM`.

The requirements are introduced incrementally as their behavior is agreed. `RTRM001` covers folder hierarchy retrieval and status counters; `RTRM002` covers on-demand account storage usage and limit; `RTRM003` covers creation of individual mailboxes and subfolders; `RTRM004` covers folder renaming and movement; `RTRM005` covers folder deletion. `RTRM006` covers filtered, sorted, paginated individual-message listings. `RTRM007` defines grouped-conversation listings incrementally; only reviewed and approved decisions are documented below. Conversation reading, searches, and remaining message operations will be specified in subsequent requirements.

## Technical requirements

### Folder inventory and status

#### RTRM001 — Retrieve folder hierarchy and status safely

Postela must obtain the complete hierarchy of available IMAP mailboxes for the authenticated account, **including unsubscribed and non-selectable folders**, without assuming a fixed namespace, hierarchy separator, or mailbox naming convention.

The backend must preserve each mailbox's full server-provided name, hierarchy separator, parent-child relationships, subscription information where provided, and attributes relevant to mailbox selectability and special-folder discovery. It must resolve effective special-folder roles according to `RTGS`, not infer them from folder names.

**Folder status retrieval:**

1. When the server supports `LIST-STATUS`, prefer `LIST` with `RETURN (STATUS (...))` to retrieve the hierarchy and the requested status values together.
2. Otherwise, retrieve the hierarchy with `LIST` and request `STATUS` individually for each selectable mailbox.
3. The requested status values are `MESSAGES`, `UNSEEN`, `UIDVALIDITY`, and `UIDNEXT`. `MESSAGES` and `UNSEEN` supply the folder's total and unread message counts. Missing status values must **never** be fabricated or interpreted as zero.
4. If `LIST-STATUS` completes but omits some requested status values for a selectable mailbox, issue a separate `STATUS` for **that mailbox** to retrieve the missing values, without repeating successful status requests for other mailboxes.

**Non-selectable mailboxes:**

- A mailbox marked `\Noselect` must remain visible in the returned hierarchy, including its descendant mailboxes.
- Its selectable-message state must explicitly indicate that it cannot be selected to read messages.
- Its `MESSAGES` and `UNSEEN` counters must be `null` rather than zero; their absence is **not itself an error**.
- Postela must not issue recovery `STATUS` requests for `\Noselect` mailboxes to fill in these counters.

**Partial `STATUS` failures:**

- Apply the already agreed connection-recovery policy: **up to three retries for temporary connection failures** affecting a status query. A definitive IMAP `NO` or `BAD` response must be reported without retrying it as a connection failure.
- If the status cannot be recovered, still return the complete hierarchy obtained by `LIST` and retain status information successfully obtained for other mailboxes.
- The affected unavailable counters or status values must be returned as `null`, accompanied by error information identifying the affected mailbox and failed query. Values successfully received must not be overwritten with fabricated zeroes.
- A status failure must not turn an otherwise successfully retrieved hierarchy into a failed folder-list operation.

**`LIST` failures:**

- Apply up to **three retries** for temporary connection failures while retrieving the hierarchy; do not retry definitive IMAP `NO` or `BAD` responses as connection failures.
- If a complete folder inventory cannot be obtained after the applicable retries, return an identifiable **operation failure**. Postela must not present a partial hierarchy as if it were complete.
- A client that already holds a previously retrieved hierarchy may keep it visible, but must identify it as **not refreshed**; this does not make the failed response a successful refresh.

Folder inventory changes and refreshes must follow the existing account-wide monitoring and special-folder resolution rules in `RTSN` and `RTGS`. This requirement does not introduce separate polling or per-client persistent IMAP monitoring connections.

**Related requirements:** RFRM001–RFRM002, RFRM005, RFRM011; RTGS011–RTGS021; RTSN003–RTSN006.

### Storage quota

#### RTRM002 — On-demand storage usage and limit through IMAP

Postela must expose only two storage-quota values to clients: **used space** (`used`) and **storage limit** (`limit`). Each value must be expressed in **bytes**, or be `null` if it cannot be established. Other IMAP quota resources, including `MESSAGE`, are outside the initial implementation scope.

**Retrieval and interpretation:**

1. Retrieve quota information **only when a client requests it**, not as part of routine folder inventory or message-list loading.
2. Start with the IMAP `GETQUOTAROOT INBOX` command. Postela must **not query every mailbox** solely to construct a full account quota overview.
3. Read the usage and limit of the `STORAGE` resource returned by the server and convert both values from IMAP's **1024-byte units** into bytes.
4. Do not treat an absent value, missing `STORAGE` resource, or lack of supported quota information as a numerical zero. Return `null` for any value that cannot be determined reliably.
5. A quota returned for `INBOX` must **not be labeled or presented as the account-wide quota unless its scope is established by the server information**. If multiple quota roots or insufficient scope information prevent establishing one account-wide used/limit pair, Postela must not choose a root arbitrarily, sum independent or overlapping roots, or invent totals. Unverifiable account-wide values remain `null`.

**Temporary account-scoped cache:**

- Once a session has requested quota information, keep the available result temporarily **per authenticated account**, shared by its authorized sessions.
- While quota information has been requested and is being tracked for that account, relevant message or mailbox changes detected through the existing account-wide IMAP monitoring, or performed successfully through Postela, must trigger a refresh of the affected quota values and notification to the account's sessions.
- Related changes may be **debounced/coalesced** to avoid issuing duplicate quota requests.
- Refresh from server-reported quota values; do **not** calculate new usage or limits by guessing the storage effects of message additions, moves, or deletions.
- No quota polling or refresh work is required for an account **before the first quota request**. The in-memory quota cache must be discarded when the account no longer has any valid sessions, following the existing account resource lifecycle.

**Failures and recovery:**

- For temporary connection failures during `GETQUOTAROOT INBOX`, apply the previously agreed policy of **up to three retries**.
- Definitive IMAP `NO` and `BAD` responses must be reported without retrying them as transient connection failures.
- If the quota request fails after the applicable retries, return **`used: null` and `limit: null`**, together with error information identifying the failed operation. A query failure must not be presented as a confirmed absence of quotas.
- If quota retrieval is unsupported or the required values are not provided, report their **unavailability** distinctly from a failed query and from a genuine zero.

**Related requirements:** RFRM003–RFRM004, RFRM063; RTCA023–RTCA025; RTSN003–RTSN006.


### Folder creation

#### RTRM003 — Create an individual IMAP mailbox or subfolder

Postela must expose an operation that creates **exactly one folder per request**. The client supplies the new folder's name and either:

- **Root placement:** create the new folder at the applicable account mailbox root.
- **Child placement:** identify an existing parent folder in which the new folder will be created.

The backend must validate that a specified parent exists in the current IMAP folder inventory. It must construct the target **full IMAP mailbox name** from the server-reported hierarchy separator, namespace, and selected parent, without assuming that the separator is `/` or automatically creating additional intermediate folders.

**Parent eligibility:**

- An existing folder with `\Noselect` may be used as a parent when the server allows children beneath it: message selectability is separate from the ability to contain subfolders.
- A folder marked `\Noinferiors` must **not** be offered or accepted as a parent.
- Other server-side restrictions still apply; the backend must not assume that a parent is writable solely because it lacks `\Noinferiors`.

**Name validation and creation:**

1. Reject an empty new-folder name.
2. Reject any new-folder name containing the hierarchy separator announced by the IMAP server, preventing one request from creating multiple hierarchy levels through an embedded separator.
3. Do not impose arbitrary additional character restrictions; defer the server-specific definitive name validation to IMAP.
4. If a mailbox with the requested full name already exists in the selected hierarchy level, return an explicit duplicate-name error. Do **not** silently rename it, append a numeric suffix, or otherwise modify the requested name.
5. Issue **one IMAP `CREATE` command** for the validated full mailbox name. Postela must not proactively create missing intermediate parent mailboxes.
6. A successfully created folder must not automatically be assigned a Postela special-folder role. The resulting inventory and any server-advertised `SPECIAL-USE` information remain subject to the existing `RTGS` special-folder resolution rules.

**Connection loss and uncertain completion:**

- A `CREATE` command modifies server state. If the connection is lost after sending it and Postela cannot determine whether the server completed it, **do not blindly repeat the command**.
- First attempt a fresh, reliable `LIST` check for the specific requested full mailbox name, validating the actual returned name rather than treating an unrelated wildcard match as confirmation.
- If the folder exists, report its current existence, **without claiming that the interrupted operation necessarily created it**.
- If a reliable check confirms the folder does not exist, Postela may retry `CREATE`, subject again to the same validation and uncertainty-handling rules.
- If the outcome cannot be verified, return an identifiable **uncertain-result error**, not an invented success or a definitive creation failure.
- Report definitive IMAP `NO` or `BAD` responses as failures rather than retrying them as connection errors.

Following a confirmed creation, refresh or propagate the affected folder inventory, counters, and relevant special-folder state to active sessions of that account using the existing shared synchronization architecture. Do not establish an additional persistent IMAP monitoring connection for this operation.

**Related requirements:** RFRM001–RFRM002, RFRM006, RFRM011; RTGS011–RTGS021; RTSN003–RTSN006.

### Folder renaming and movement

#### RTRM004 — Rename or move existing IMAP folders

Postela must expose **two distinct operations**, `rename_folder` and `move_folder`, both implemented using the IMAP `RENAME` command:

- **Rename:** replace the selected folder's final name component while retaining the existing parent and namespace.
- **Move:** keep the selected folder's final name component while changing its parent, or moving it to the applicable mailbox root.

The backend must use the full IMAP folder names and hierarchy separators obtained from the server. Both operations must preserve the original mailbox's messages and descendant hierarchy as the server allows. The server's `RENAME` semantics, not a sequence of client-side folder-copy operations, are authoritative.

**Validation before `RENAME`:**

1. The source must be an existing folder. **`INBOX` itself must never be renamed or moved**, even if the server permits it; its subfolders may be renamed or moved where eligible.
2. For a move to a parent folder, verify that the destination parent exists. Allow an existing `\Noselect` parent when the server permits subfolders, but reject a parent with `\Noinferiors`. Other server restrictions remain authoritative.
3. Reject moving a folder into **itself or any of its descendants**, preventing circular or self-nesting hierarchies.
4. Reject a target full mailbox name already occupied by **another folder**. Do not overwrite, merge, or silently generate an alternative name. The same restriction applies to conflicting descendants where they can be determined before execution.
5. Reject invalid requested final folder names under the single-component naming restrictions established for creation in RTRM003, including empty names and embedded hierarchy separators. The server performs definitive validation.
6. If the requested name and parent already match the current state, return **success with no changes** and do not issue `RENAME`. This is distinct from a destination occupied by another folder.

**Connection loss and uncertain outcome:**

- Because `RENAME` changes server state, do **not** blindly repeat it after a lost connection when its completion is unknown.
- Obtain a reliable refreshed `LIST` inventory covering the source, target, and affected descendants. Check full mailbox names and hierarchy relationships to distinguish a completed move/rename from no change or a partial/uncertain result.
- If the entire expected rename or move is confirmed, treat the IMAP change as completed and proceed with the special-folder mapping updates described below.
- If the inventory reliably confirms that **no change occurred**, Postela may retry `RENAME`, subject to the same validation and failure-handling safeguards.
- If the observed result is **partial or uncertain**, return an error with the verified state and do not attempt additional modifications or automatic rollback. Definitive IMAP `NO` or `BAD` responses must be reported as failures, not automatically retried as connection errors.

**Preserve account special-folder assignments:**

Following a confirmed IMAP rename or move, Postela must determine which explicit special-folder mappings refer to the renamed/moved folder or descendants whose full names have changed.

- For existing **user overrides**, update each affected full mailbox name in the account's `special_folders` preference while preserving unrelated assignments.
- For affected **administrator-defined mappings**, leave TOML unchanged and create the appropriate account-specific user override with the new full mailbox name.
- For **automatically discovered `SPECIAL-USE` mappings**, refresh discovery from the server; do not create an unnecessary explicit override.
- The account's effective mappings must continue to respect the precedence and validation rules in RTGS. Changes to the `special_folders` preference must use the agreed SQLite atomic-update semantics.

**If SQLite updating the mappings fails after successful IMAP `RENAME`:**

1. **Do not automatically reverse or repeat the successful IMAP rename/move.**
2. Report the **partial success** explicitly: the folder was renamed or moved, but special-folder mappings could not be updated. Include appropriate error information.
3. Preserve enough source/destination and affected-mapping information to retry the **mapping update only**, without executing `RENAME` again. Do not silently mark the saved mappings as corrected when the SQLite transaction failed.
4. Refresh the actual folder hierarchy and special-folder availability and notify other active sessions of the same account of the real observed state, distinguishing committed mappings from unavailable/stale ones.

All operation failures must follow RTCA036–RTCA037's cross-cutting structured logging and sensitive-data protections. Following successful or partially successful server changes, use the existing shared IMAP monitoring and session notification mechanisms rather than creating additional persistent connections.

**Related requirements:** RFRM007–RFRM011; RTGS013–RTGS021, RTGS027; RTCA036–RTCA037; RTSN003–RTSN006.

### Folder deletion

#### RTRM005 — Delete IMAP folders under administrator policies

Postela must support deleting a single existing IMAP folder, optionally including its descendants, under the three administrator-controlled policies defined in RTGS027:

- `allow_delete_nonempty_folders` (default `false`): whether to empty mailboxes containing messages before deleting them.
- `allow_recursive_folder_deletion` (default `false`): whether to delete descendant mailboxes.
- `allow_delete_special_folders` (default `false`): whether to delete folders resolved as special-use roles.

**`INBOX` itself must never be deleted** regardless of these settings. Subfolders of INBOX may be deleted when independently permitted. IMAP `DELETE` does not delete descendant mailboxes; recursive deletion must process **deepest descendants first, parent last**, using separate IMAP operations.

**Full preflight before any modification:**

1. Obtain a sufficiently current server folder hierarchy, including descendants, and determine the scope of the requested operation. If descendants exist but `allow_recursive_folder_deletion = false`, reject the operation without deleting anything.
2. Identify all affected special-folder roles. If `allow_delete_special_folders = false` and any affected mailbox is special, or if the operation targets `INBOX` itself, **reject the complete operation** and identify the blocking mailboxes before changing anything.
3. Retrieve `MESSAGES` via `STATUS` for each affected selectable mailbox where the count is needed for validation or emptying. Non-selectable `\Noselect` hierarchy nodes do not require message-count recovery solely to determine whether they contain selectable messages.
4. If `allow_delete_nonempty_folders = false` and any affected mailbox has messages, reject the whole operation and identify the nonempty mailboxes. If any required status is unavailable, **do not assume a `null` count means empty**.
5. On temporary connection failures during required `STATUS` queries, apply **up to three retries**. Definitive IMAP `NO` or `BAD` responses are errors, not transient connection failures. If the count still cannot be reliably determined, reject the **entire operation before emptying or deleting any folder** and return the affected mailbox and error.
6. Resolve the effective `delete_to_trash` setting and Trash mapping. If `delete_to_trash = true` and the currently configured Trash mailbox is **within the deletion subtree**, reject the entire operation as incompatible, even when `allow_delete_special_folders = true` and even when the requested folder is Trash itself. To permit deletion, the user must first choose an existing Trash mailbox **outside** the subtree. Postela must never remap Trash automatically.
7. Where moving messages to Trash will be required, validate that the resolved Trash destination is available before beginning. If not, reject rather than silently switching to permanent deletion.

These validations are not an isolation guarantee against concurrent changes by another IMAP client. The backend must respect actual IMAP permissions and the current results of each operation.

**Emptying before folder deletion:**

- If `allow_delete_nonempty_folders = true`, empty each affected mailbox containing messages before executing IMAP `DELETE` for that mailbox, following `delete_to_trash`.
- With `delete_to_trash = true`, move messages into the configured Trash mailbox. With `delete_to_trash = false`, permanently delete messages.
- Messages deleted **from Trash itself** follow the pre-existing permanent-delete rule when deleting that folder is permitted under the effective settings; this does not bypass the incompatibility check above.
- A failure during emptying must **stop the entire remaining operation**, leaving the affected folder undeleted, and report any message changes already completed and the error. Do not automatically reverse completed moves or deletions.
- Do not empty descendants when performing a nonrecursive folder deletion.

**Execution and partial results:**

- After preflight, process folders deepest-first. A completed deletion of one child is not rolled back when a later child fails.
- At the **first failure** to empty or delete a folder, stop immediately; do not process remaining siblings or ancestors.
- Report separately the mailboxes confirmed deleted, any confirmed message changes before failure, the step where the operation stopped, and the error. Never claim that a partially successful recursive deletion was atomic or complete.
- Refresh the actual server folder inventory and special-folder availability, and notify affected sessions of the same account through the existing IMAP monitoring architecture.

**Unknown outcome after connection loss during IMAP `DELETE`:**

1. Do **not** automatically repeat a `DELETE` command when its outcome is uncertain.
2. Obtain a reliable fresh `LIST` inventory for the affected mailbox and hierarchy.
3. If the mailbox no longer exists, record the deletion as confirmed; do not attempt to delete it again.
4. If it still exists, verify its actual current state and permissions before a possible retry; its continued presence alone does not prove the previous attempt had no effects.
5. If a reliable determination is impossible, return an identifiable **uncertain-result error** and stop all remaining destructive operations.

**Special-folder mappings after confirmed deletions:**

- When Postela confirms deletion of a mailbox assigned to one or more special-folder roles, explicitly set every affected role to `null` in the account-wide `account_preferences.special_folders` preference. This applies to affected descendants in recursive deletions as well.
- An **explicit `null`** means that the role is unassigned and takes precedence over administrator-defined mappings and IMAP `SPECIAL-USE` discovery. An **absent property** means there is no user override, so the usual RTGS resolution rules apply. Do not treat these states as interchangeable.
- Do not automatically choose, create, or assign a replacement special mailbox. Preserve all unaffected roles and preferences.
- Only update mappings for mailboxes **confirmed deleted**. In a partial recursive deletion, do not clear mappings for mailboxes still present or whose removal could not be verified. Use the established account-wide partial-update and SQLite transaction semantics.
- Notify all active sessions of the affected account about the confirmed folder changes and the resulting special-folder availability.

**SQLite failure after a successful IMAP deletion:**

1. Do **not** attempt to restore or repeat a mailbox deletion that has already succeeded on the IMAP server.
2. Record the SQLite failure as an `ERROR` under RTCA036–RTCA037, and report **partial success**: the mailbox was deleted, but its special-folder preferences could not be updated.
3. Preserve sufficient information about the confirmed deletions and affected roles to retry **only the SQLite preference update**, without executing IMAP `DELETE` again.
4. Refresh the actual folder hierarchy and notify account sessions of the server changes, distinguishing the confirmed IMAP result from stale or uncommitted preference mappings.
5. For recursive operations, update or retry mappings **only for mailboxes whose deletion has been confirmed**, regardless of whether other deletions failed or remain uncertain.

Failures and incomplete results must be logged in accordance with RTCA036–RTCA037, without exposing credentials or message content. This requirement does not create an additional per-client IMAP monitoring connection.

**Related requirements:** RFRM010–RFRM011, RFRM035–RFRM037; RTGS014–RTGS022, RTGS027; RTCA036–RTCA037; RTSN003–RTSN006.

### Individual-message listing, filtering, and pagination

#### RTRM006 — Server-side filtering, sorting, and paginated individual-message listings

When the effective `thread_view` preference is `false`, or conversation grouping is unavailable, Postela must retrieve **individual-message** folder listings using the server's IMAP search and sorting facilities wherever supported. **Filtering precedes sorting, and sorting precedes pagination.** The backend must not download all messages or all their metadata merely to construct a page.

**Folder selection and effective options:**

1. Select or examine the requested selectable mailbox using a non-persistent, on-demand IMAP connection. Obtain its current `UIDVALIDITY` and validate it against any cached UID results for that folder.
2. Determine the effective `page_size` and `sort` from `RTGS`, subject to valid temporary request overrides. Changing the listing's sorting or quick filters must not modify `account_preferences`.
3. Consult the account's existing IMAP capability cache under `RTCA`; do not issue a fresh `CAPABILITY` before every page request solely for this purpose.
4. Expose the effective available sorting criteria, direction, and any fallback to the client through the agreed capability and response contract, without exposing raw IMAP extension names as product-level features.

**Folder-local quick filters:**

- No quick filter: use the equivalent of IMAP `ALL`.
- Unread: use IMAP `UNSEEN`.
- Flagged: use IMAP `FLAGGED`.
- Unread **and** flagged: combine the IMAP search keys with **AND** semantics (`UNSEEN FLAGGED`).
- Apply these criteria **only to the selected mailbox**, never to its descendants or to related messages in other folders, irrespective of `thread_multifolder`.
- Supply the search criteria directly to `SORT` when sorting is available; otherwise use `UID SEARCH`. Do not fetch complete messages or evaluate these filters against downloaded message bodies.

**Server-side sorting and pagination strategy:**

1. **`CONTEXT=SORT` available:** prefer extended `UID SORT` with `RETURN (PARTIAL start:end)`, passing the sorting criteria and IMAP search keys in the same operation. Retrieve only the UID interval for the requested page from the server; no Postela ordered-UID result cache is required for this strategy.
2. **`SORT` available without `CONTEXT=SORT`:** use `UID SORT` with the applicable search keys so the server performs both filtering and sorting. This returns the complete ordered UID result; Postela extracts the requested page's UID range. Reuse the session's ordered-UID cache as defined below.
3. **No usable `SORT`:** apply quick filters with `UID SEARCH`, paginate the server-provided results, and **do not sort locally**. Inform the client that the requested sorting is unavailable. In particular, do not retrieve an entire folder's metadata to emulate sorting.
4. `ESORT` by itself does **not** establish support for `PARTIAL` on sorted results; choose the partial sorted-result strategy only when the required capability is actually available.
5. Apply the previously agreed sorting fallbacks: if `sender_name` is unavailable, prefer `sender_email` with the same direction; for other unavailable criteria, prefer date descending and then another server-supported criterion if applicable. If no usable sorting criterion exists, use the unsorted server-provided results. Preserve the saved preference during any temporary fallback.
6. A page must always be determined from the **entire matching result in the effective order**, not by sorting only the messages fetched for that page. If the selected criteria return no UIDs, return an empty page rather than making an invalid `FETCH` request.

**Ordered-UID cache (only for the `UID SORT` fallback):**

- Maintain **at most one ordered-UID cache per authenticated session**, independent of other sessions of the same account. Store UIDs as `[]uint32`, not message bodies, metadata, or a persistent message index.
- The cache represents a specific mailbox, its `UIDVALIDITY`, effective sorting criterion and direction, and active quick-filter combination. Replace it when the session requests a different mailbox, sorting, or filtering; do not reuse UID results for a different query.
- Use the administrator-configurable capacity defined in `RTGS007`, with a Postela default of **200,000 UIDs per session**. If a `UID SORT` result exceeds that limit, resolve the current page from the full returned UID list and then discard it **without retaining a truncated cache**. Later pages must remain accessible and may require repeating `UID SORT`.
- Invalidate the affected session caches on relevant IMAP notifications, including message arrival, expunge or movement, changes that can affect filtering or ordering, and a changed `UIDVALIDITY`. Do not invalidate unrelated account sessions' caches unless their cached results are also affected.
- **Regenerate only on demand:** invalidation does not immediately launch another `UID SORT`. Rebuild the applicable cache the next time that session requests a listing.
- Keep the cache across transient WebSocket reconnections while the backend session remains valid. Release it immediately on logout or any session invalidation, including reconnection-period expiration and backend restart. Follow the existing session lifecycle; no additional persistent IMAP connection is introduced.

**Metadata retrieval and response identity:**

1. Once the page's UIDs are known, retrieve their listing metadata using **one `UID FETCH` for that page**, requesting `UID`, `FLAGS`, `ENVELOPE`, `INTERNALDATE`, `RFC822.SIZE`, and `BODYSTRUCTURE`, as applicable.
2. Use the MIME structure to identify downloadable attachments without downloading attachment contents or message bodies. This listing operation must not itself mark messages as read.
3. Preserve the effective page order when assembling the response; do not assume that the order of individual `FETCH` responses necessarily matches the sorted UID sequence.
4. Include the full folder name and its **current `UIDVALIDITY` once at response level**, plus the message UID in each entry, rather than duplicating the folder identity in every entry. Provide the information needed to navigate pages and distinguish effective sorting/filtering from requested preferences.
5. If `UIDVALIDITY` changes while resolving a page, do not present cached results from the previous UID generation as valid. Follow the existing safe-identity and failure/recovery rules.

The grouped-conversation listing (`thread_view = true`) has separate threading, grouping, representative-message, aggregate-counter, sorting, and pagination requirements. It must not be implemented by simply relabeling this individual-message list.

**Related requirements:** RFRM012–RFRM016, RFRM053–RFRM060, RFRM062, RFRM064; RTGS003–RTGS007; RTCA session/capability-cache rules; RTSN003–RTSN006.

### Grouped conversation listings

#### RTRM007 — Grouped conversation structure and folder-local quick filters

When the effective account preference is `thread_view = true`, Postela must use **`UID THREAD REFS`** on the selected IMAP mailbox to obtain the server-provided conversation structure.

- Retrieve the **complete conversation structure of the selected folder**, including messages that do not match the currently active quick filters. Quick-filter application must not determine the initial membership of a conversation.
- Return at most one grouped entry for each conversation represented in that folder. A conversation may also be represented independently in the listing of another folder in which it has physical messages.
- If the IMAP server does **not** advertise `THREAD=REFS`, use the individual-message listing described in `RTRM006`. Preserve the saved `thread_view` preference and inform the client of the unavailable grouped-list capability.
- The conversation structure **may be cached temporarily per authenticated session** within the separate limits already defined in `RTGS025`, without persisting a message/conversation index. Detailed cache lifecycle and invalidation rules will be specified in later decisions for this requirement.

**Folder-local quick filtering:**

1. Apply quick filters **only to messages physically present in the currently selected IMAP mailbox**, excluding its subfolders and all other mailboxes, regardless of `thread_multifolder`.
2. Use IMAP `UID SEARCH UNSEEN` for unread messages, `UID SEARCH FLAGGED` for flagged messages, and `UID SEARCH UNSEEN FLAGGED` when both filters are active (**AND** semantics). When no quick filter is active, every conversation in the selected folder is eligible.
3. Intersect the returned matching UIDs with the members of each conversation in the complete `UID THREAD REFS` structure for that folder, reusing the cached structure if available. **Do not rebuild the thread structure by running `THREAD REFS` only on filtered messages.**
4. Include a conversation in the filtered listing if **at least one of its messages belonging to the currently selected folder** matches every active filter. Do not search other folders to determine whether a conversation qualifies.
5. Filtering must not split a conversation, change its complete membership, or reduce its reported conversation counters to the subset of matching messages. Technical rules for counter calculation and representative selection are specified separately.

**Pending decisions within RTRM007:** selection of representative messages, grouped sorting and pagination, conversation-counter computation, and detailed cache behavior. These aspects will be defined through separately reviewed technical decisions.

**Related requirements:** RFRM012, RFRM014–RFRM015, RFRM053–RFRM054, RFRM057; RTGS025; RTRM006.

## Scope boundaries and follow-up documentation

- **Quota information:** `RTRM002` implements only on-demand `STORAGE` usage and limit, beginning with `GETQUOTAROOT INBOX`. A complete per-mailbox quota-root inventory and other quota resources are outside the initial scope.
- **Folder modifications and listings:** `RTRM003` covers folder creation, `RTRM004` covers renaming and movement, `RTRM005` covers deletion, and `RTRM006` covers individual-message listings. `RTRM007` is being defined incrementally for grouped-conversation listings; remaining message operations will follow.
- **Server authority:** actual mailbox inventory, status values, attributes, and access restrictions come from the IMAP server.
