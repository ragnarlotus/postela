# Reading and management

**Group:** `RTRM`

**Numbering:** `RTRM001`, `RTRM002`, `RTRM003`…

**Related requirements:** [Functional reading and management](../functional/reading-and-management.md) (`RFRM`), [Technical general settings](./general-settings.md) (`RTGS`), [Core architecture](./core-architecture.md) (`RTCA`), and [Sync and notifications](./sync-and-notifications.md) (`RTSN`).

## Scope

This document specifies the technical implementation of mailbox reading and management through IMAP, following the functional requirements in `RFRM`.

The requirements are introduced incrementally as their behavior is agreed. `RTRM001` covers folder hierarchy retrieval and status counters; `RTRM002` covers on-demand account storage usage and limit; `RTRM003` covers creation of individual mailboxes and subfolders; `RTRM004` covers folder renaming and movement; `RTRM005` covers folder deletion. `RTRM006` covers filtered, sorted, paginated individual-message listings. `RTRM007` covers grouped-conversation listings and their session cache. `RTRM008` covers individual-message reading, headers, and downloads. `RTRM009` covers conversation reading. `RTRM010` defines explicit IMAP flag modifications incrementally, recording only reviewed and approved decisions below. Searches and remaining message operations will be specified in subsequent requirements.

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

**Representative message when the IMAP server supports `SORT`:**

1. Determine each conversation's representative **only from its messages in the currently selected folder**. With no quick filters, select its most recent message; with `UNSEEN`, `FLAGGED`, or both, select its most recent message that satisfies all active filters.
2. Obtain the server's message-date ordering through `UID SORT` and cross-reference the resulting ordered UIDs with the complete folder-local `UID THREAD REFS` conversation structure. Where quick filters are active, use the folder-local matching UID set already obtained via `UID SEARCH` when selecting a representative.
3. Select the first eligible UID in the resulting most-recent-first order for each conversation. **Do not infer message recency from the numerically largest UID** and do not fetch metadata for the entire folder solely to choose representatives.
4. This representative-selection ordering is **independent of the user's requested sorting criterion for the conversation listing**. Grouped-list sorting and pagination are separate decisions.

**Representative message when the IMAP server lacks `SORT`:**

1. Obtain the full folder-local conversation structure through `UID THREAD REFS` and apply the approved folder-local quick filters.
2. **Do not sort the complete folder locally** or retrieve metadata for every message to emulate missing IMAP sorting. For the unsorted fallback, retain the conversation order provided by IMAP and determine the requested page of conversation entries first.
3. Only for conversations on that page, use `UID FETCH` to obtain the necessary message dates of their eligible member UIDs. When quick filters are active, restrict candidates for representative selection to members matching every active filter.
4. Choose the most recent eligible message in the selected folder as the representative of each listed conversation. This fallback must not query messages in other folders merely to choose representatives.
5. Keep this date retrieval limited to the page's conversations; do not download full message bodies or attachments. The backend must not claim that the **conversation listing itself** is sorted when the server supplies no usable sorting capability.

**Grouped-conversation sorting and pagination when `SORT` is available:**

1. Obtain or reuse the complete folder-local `UID THREAD REFS` conversation structure, apply approved folder-local quick filters, and determine one eligible representative UID for each matching conversation using the representative-selection rules above.
2. Use IMAP **`UID SORT`** to request message UIDs in the effective sorting criterion and direction chosen by the client, applying the previously agreed available-criterion fallbacks. The IMAP server performs the ordering; Postela must not retrieve every message's metadata to sort locally.
3. Cross-reference the server-ordered UIDs with the **representative UID of each eligible conversation**, retaining each representative only once to form an ordered list of **conversations**, not individual messages.
4. Apply `page_size` and the requested page to that ordered list of conversations **after** folder-local filtering and conversation ordering. `page_size` therefore counts conversations when `thread_view = true`.
5. Fetch only the listing metadata needed for the representative messages of the requested conversation page, without fetching full bodies or attachments. Preserve the computed conversation order in the response.
6. **Do not use `CONTEXT=SORT` `PARTIAL` directly to page conversations:** its range refers to message positions, which do not correspond to conversation positions. Even when partial results are supported, the grouping and deduplication of representative UIDs must precede pagination.

**Folder-local conversation counters (`thread_multifolder = false`):**

1. Use the complete conversation membership returned by `UID THREAD REFS` for the selected mailbox (or the valid cached structure) to determine the physical UID members of each conversation.
2. For read and flagged state, request `UID SEARCH UNSEEN` and `UID SEARCH FLAGGED` in **that same mailbox**. Reuse compatible search results already retrieved for folder-local quick filters, rather than repeating queries unnecessarily.
3. Compute `thread.count` as the number of member UIDs in the conversation's folder-local structure; compute `thread.unread` and `thread.flagged` by intersecting those member UIDs with the respective `UNSEEN` and `FLAGGED` search results.
4. Calculate and include these counters **only for conversations on the requested conversation page**, even when an IMAP search returns matching UIDs from the whole selected mailbox.
5. These counters cover **all member messages physically in the selected folder**, regardless of which messages matched the active quick filters. Do not include related messages from other folders when `thread_multifolder = false`.
6. Do not use per-message `FETCH` solely to count unread or flagged messages; rely on the IMAP search results and the conversation structure.

**Cross-folder conversation counters (`thread_multifolder = true`):**

1. Determine the conversations included in the requested page using the **selected folder's** complete `UID THREAD REFS` structure, quick filters, representative selection, sorting, and pagination as agreed above. Do not expand the folder-local quick-filter scope or change which conversations appear on that page.
2. **Only for conversations on that page**, discover their related physical messages in other available account folders on demand, using explicit `Message-ID`, `In-Reply-To`, and `References` relationships. Do not treat a matching subject or sender alone as proof of conversation membership.
3. Track each discovered physical message with its source folder, `UIDVALIDITY`, and UID. Distinct physical copies sharing a `Message-ID` must still be counted separately.
4. Obtain the necessary read and flagged state from IMAP without downloading message bodies or attachments. Derive `thread.count` from the number of discovered physical messages, `thread.unread` from messages without `\\Seen`, and `thread.flagged` from those with `\\Flagged`.
5. If querying another folder fails, **preserve the folder-local listing and quick filters**. Apply the existing transient-connection retry policy. Return `null` for any affected aggregate counter that cannot be established reliably, together with identifiable error information; never present a partial value as an exact count. Counters that remain reliable may still be returned.
6. Do not persist a local message or conversation index. Cross-folder discovery is performed only for the page's conversations, not for conversations on other pages.

**Conversation-structure cache lifecycle:**

1. **Create on demand:** when a session first requests a grouped listing for a folder, obtain its complete `UID THREAD REFS` structure and retain it for potential reuse, subject to the cache capacity defined in `RTGS025` (Postela default: **200,000 member UIDs per session**, administrator-configurable through TOML).
2. **Reuse:** preserve the same folder's structure when requesting other pages, applying different folder-local quick filters, or changing the conversation-list sort order. These changes do not, by themselves, modify the underlying conversation membership.
3. **Switch folders:** hold the structure for **at most one selected folder per session**. Discard the previous folder's cached structure when another folder's structure is needed. Keep caches separate for different authenticated sessions.
4. **Invalidate on membership changes:** invalidate the affected folder's cached structure after message arrival, expunge/deletion, or movement that changes its membership, or when `UIDVALIDITY` changes. Flag-only changes (`\\Seen`, `\\Flagged`) do not invalidate conversation relationships; refresh or invalidate the affected unread/flagged counters separately.
5. **Regenerate only on demand:** an invalidation must not immediately execute another `UID THREAD REFS`. Rebuild when the session next requests a grouped listing that needs the structure. Avoid duplicate concurrent reconstructions for the same session and folder.
6. **Handle disconnections:** retain a valid structure across transient client WebSocket disconnections while the backend session remains valid. Invalidate it if IMAP monitoring is interrupted and changes could have been missed.
7. **Release:** discard the cache on explicit logout, session expiration or invalidation, and backend restart, following the established session lifecycle.
8. **Exceeding capacity:** if a folder's complete thread structure exceeds the configured UID limit, process the requested page using that result, then discard the structure without retaining a truncated cache. Exceeding cache capacity must never prevent accessing a large folder or artificially reduce `page_size`.

This cache is temporary and session-scoped: it does not persist a message or conversation index and does not create a per-client persistent IMAP connection.



**Related requirements:** RFRM012, RFRM014–RFRM015, RFRM053–RFRM054, RFRM057; RTGS025; RTRM006.

### Individual-message reading

#### RTRM008 — Identify and fetch one message safely

The `read_message` operation must retrieve **only the message explicitly requested** by the client, independently of the account's `thread_view` preference:

1. Accept the selected message's **folder, `UIDVALIDITY`, and UID** as its complete identity.
2. Open/select the specified mailbox through an on-demand authenticated IMAP connection, following the existing connection and session-management rules.
3. Check the mailbox's current `UIDVALIDITY` against the client's supplied value **before attempting to fetch or modify the message**.
4. On a `UIDVALIDITY` mismatch, return a specific message-identity error and do not attempt to retrieve the UID from the changed mailbox.
5. When identity matches, use `UID FETCH` targeted to **that UID only** to retrieve the requested message.
6. If the specified UID no longer exists in the mailbox, return a specific message-unavailable error.
7. Do **not** discover related messages or perform conversation-thread searches as part of `read_message`. Conversation reading belongs to the separate `read_thread` operation.

**Initial metadata and MIME-structure retrieval:**

1. After validating the mailbox's `UIDVALIDITY`, use a `UID FETCH` request targeted to the selected message to obtain `ENVELOPE`, `FLAGS`, `INTERNALDATE`, and `BODYSTRUCTURE`.
2. Use `ENVELOPE` for the available sender, recipient, subject, and message-date information; `FLAGS` for the current IMAP flags; and `INTERNALDATE` for the date recorded by the IMAP server.
3. Use `BODYSTRUCTURE` to determine the message's MIME hierarchy, available content types, attachment filenames and sizes where provided, and embedded-part `Content-ID` values. Expose metadata only when the underlying message provides it; do not invent missing names or sizes.
4. Do not download the contents of every attachment in order to construct the attachment listing. The exact body-section retrieval and individual-attachment download rules will be specified separately.

**HTML and plain-text MIME body retrieval:**

1. Use the already-fetched `BODYSTRUCTURE` hierarchy to identify available `text/html` and `text/plain` message body parts, respecting MIME nesting and `multipart/alternative` relationships.
2. Retrieve **only the required message-body sections** through targeted `UID FETCH BODY.PEEK[section]` requests. Do not fetch full downloadable attachments merely to render the message body.
3. Decode each retrieved part's `Content-Transfer-Encoding` and convert text into UTF-8 when its source character set is known.
4. Return the available HTML and/or plain-text representation to the client. Respect the effective account preference `RFGS010` (HTML by default) for the preferred display format; if that representation is absent, use the available safe alternative under `RFGS011`.
5. Use `BODY.PEEK` rather than a non-PEEK body fetch so that retrieving message content **does not implicitly set `\\Seen`**. Any automatic change to read state must occur through the separate, preference-controlled `mark_as_read` behavior.
6. Sanitization of HTML before returning it to the client is defined below. Authorization/delivery of remote or CID-embedded resources is specified separately; choosing HTML does not grant permission to load unsafe active or remote content.

**Automatic marking as read (`mark_as_read`):**

1. Retrieve the requested message content using `BODY.PEEK`, preventing message-body retrieval from implicitly changing the IMAP `\\Seen` flag.
2. Check the effective account-wide `mark_as_read` preference (enabled by default under `RFGS007`) and the message's current `\\Seen` state obtained from `FLAGS`.
3. If `mark_as_read = true` and the selected message is unread, explicitly set only its read flag using `UID STORE +FLAGS.SILENT (\\Seen)`, targeting the same UID after the required identity check. Preserve every other message flag.
4. If `mark_as_read = false` or the selected message is already read, do not perform a flag change.
5. Reflect a successfully updated read state in the operation result and propagate the change through the existing account-authorized session notifications and IMAP synchronization mechanism.
6. If the flag update fails, **return the message content already retrieved**, report the flag-update error separately, and do not claim the message was marked as read. Do not treat retrieving the content and changing its read flag as an inseparable operation.

**HTML sanitization and client-side resource authorization:**

1. Sanitize decoded HTML email **in the backend before returning it to clients**, without modifying the original RFC 5322/MIME message stored on the IMAP server.
2. Remove executable or unsafe elements, including `<script>`, `<iframe>`, and `<object>`, and remove JavaScript event-handler attributes such as `onclick` and `onload`.
3. Reject dangerous URL schemes, including `javascript:`, and prevent HTML attributes, styles, or embedded content from executing scripts or bypassing the email-content security restrictions.
4. Preserve useful visual formatting where safe, including tables, hyperlinks, images, colors, and supported styles. Visual fidelity must not override the safety restrictions.
5. The frontend controls whether externally hosted resources are loaded, following the existing default-blocking and trusted-sender policy. **Sanitization must not itself authorize remote resource loading**, and trusting a sender must not authorize active content.
6. Preserve access to the unmodified original message for the separate `.eml` download capability; sanitization applies only to the display representation.

**Inline CID images in the message response:**

1. Identify `cid:` image references in the sanitized HTML and resolve each referenced `Content-ID` against the message's existing `BODYSTRUCTURE` MIME hierarchy.
2. Retrieve **only the referenced inline image MIME sections** from IMAP, using `UID FETCH BODY.PEEK[section]`, alongside the message content within the same `read_message` operation. Do not proactively download unrelated downloadable attachments or unreferenced inline parts.
3. Decode each required MIME part, validate that its actual content type is an allowed passive image format, and exclude potentially active or unsafe formats such as SVG from direct embedding.
4. Encode each accepted image as Base64 and **replace its corresponding `cid:` reference in the sanitized HTML with a `data:image/...;base64,...` URL**. Return the resulting HTML in the single `read_message` response; the frontend need not create `Blob` URLs or make additional backend requests for these inline images.
5. The client-side content-security policy for displaying mail must permit the necessary **image-only `data:` URLs** without permitting active embedded content or automatically authorizing externally hosted resources.
6. Do not persist these extracted images or modify the original message stored on IMAP. The separate `.eml` download continues to preserve the original content.

**Preventive blocking of remote resources:**

1. During backend HTML sanitization, identify externally hosted resources whose references may trigger automatic browser requests, including image sources, CSS background resources, and `srcset` entries.
2. Replace these fetch-triggering references with **inert placeholders**, retaining the original URLs as associated resource metadata. The HTML initially returned to the frontend must not contain active externally hosted references that could be loaded automatically.
3. The frontend initially renders the sanitized message **without activating remote references**, so displaying the message cannot accidentally initiate remote-resource downloads.
4. Only when the sender satisfies the **trusted-sender and DMARC authorization conditions specified below**, or when the user explicitly authorizes resources for that message, may the frontend restore the **specifically allowed remote references** for browser loading.
5. Inline CID images already embedded as validated `data:image/...;base64,...` URLs remain available immediately and are independent of the remote-sender trust decision.
6. Ordinary clickable links such as `<a href="https://...">` may remain available after sanitization because merely displaying them does not trigger automatic loading. Active content and dangerous URL schemes remain prohibited regardless of sender trust.

**Trusted-sender authorization for remote resources:**

1. Automatically authorize otherwise permitted remote resources only when the message's `From` email address matches an **exact address** in the authenticated account's trusted-sender list. A matching domain alone is insufficient.
2. In addition to the address match, require a **reliable DMARC `pass` result aligned with the `From` domain** when the receiving mail infrastructure provides trustworthy authentication results. A matching `From` address without such verification must not authorize automatic remote-content loading.
3. Under the centralized mail-server configuration defined by `RTCA009`, **the receiving mail infrastructure configured by the administrator is trusted**. Postela delegates SPF, DKIM, and DMARC verification to that infrastructure and consumes its authenticated results rather than repeating the cryptographic verification itself.
4. The configured receiving infrastructure **must strip or neutralize forged authentication-result headers** before recording its own results. Merely finding an `Authentication-Results` header in stored email does not establish that it originated from the receiving infrastructure; Postela must not treat unverifiable results as reliable. This is an **infrastructure security requirement**, not an additional list of trusted servers to maintain in Postela.
5. **Do not add `trusted_authserv_ids` configuration or a separate administrator-managed authentication-server allowlist.** The existing administrator-managed IMAP/server configuration is the trust boundary; no Postela-specific DKIM re-verification of full message bodies is required.
6. If the address is unlisted, authentication fails, or the DMARC result cannot be reliably attributed to the configured receiving infrastructure, **keep remote resources blocked automatically**. A listed address does not prove that the individual sender authored the message; DMARC authenticates aligned domains.
7. When automatic authorization cannot be established, allow the user to explicitly authorize remote content **for the currently displayed message only**, without adding or changing any trusted-sender entry.
8. Whether loading is automatic or manually authorized, only restore remote URLs otherwise permitted by the approved sanitization and resource-blocking rules. Never allow scripts, active content, or unsafe URL schemes.

**Complete original headers (`read_headers`):**

1. Provide a distinct `read_headers` operation accepting the message's **folder, `UIDVALIDITY`, and UID**.
2. Select the specified folder and validate its current `UIDVALIDITY` before reading; apply the same safe message-identity and missing-UID error handling as `read_message`.
3. Retrieve only the original header section using a UID-targeted `UID FETCH BODY.PEEK[HEADER]` operation.
4. Return the **complete original header text as received**, preserving header order, folding, and content without filtering, reconstructing, sanitizing, or rewriting it. Include any present `Received`, `Authentication-Results`, `DKIM-Signature`, and other headers.
5. Do not download the message body or attachments. `read_headers` must never alter `\\Seen`, regardless of the account's `mark_as_read` preference.
6. The client may display or copy the returned headers **as text**, never interpreting them as active HTML.

**Download the unmodified original message (`.eml`):**

1. Provide a separate original-message download operation that accepts the message's **folder, `UIDVALIDITY`, and UID**, and validates the selected mailbox's current `UIDVALIDITY` before accessing the UID, using the approved missing-message error behavior.
2. Retrieve only that message's **complete original RFC 5322/MIME representation**, including every header, MIME boundary, body part, and attachment, using a UID-targeted `UID FETCH BODY.PEEK[]`.
3. Deliver the original bytes to the requesting client **without modifying** headers, MIME encoding, boundaries, or contents. Prefer incremental/streamed transfer when supported by the selected client transport, so large messages need not be retained fully in application memory; the transport-level transfer mechanism will be specified separately if needed.
4. `BODY.PEEK[]` must not implicitly mark the message as read, and this download must **not** apply the `mark_as_read` preference or otherwise change `\\Seen`.
5. Allow the client to choose a safe download filename with the `.eml` extension. Do not store a persistent copy in SQLite or persistent temporary files.
6. Keep this operation independent of `read_message`, so simply opening a message does not require fetching the entire raw message and all of its attachments.

**Download one selected attachment on demand:**

1. During `read_message`, use the retrieved `BODYSTRUCTURE` to identify downloadable attachment parts and provide the client with their MIME section identifiers and available attachment metadata.
2. The client requests a specific attachment by supplying its **folder, `UIDVALIDITY`, UID, and MIME section identifier**.
3. Select the mailbox, validate its current `UIDVALIDITY` against the supplied identity, confirm that the message still exists, and **verify against its current MIME structure** that the requested section identifies a downloadable attachment of that message. Reject missing, invalid, or non-attachment sections.
4. Retrieve **only that MIME part** using a UID-targeted `UID FETCH BODY.PEEK[section]`. Do not fetch other attachments or the entire original message.
5. Decode the part's `Content-Transfer-Encoding`, including Base64 or quoted-printable when applicable, to recover the attachment's original file bytes.
6. Return the attachment bytes with its validated MIME type and its original filename where safely usable; sanitize filenames to prevent unsafe paths or names. The frontend handles download presentation and optional preview.
7. This operation must not modify `\Seen` or apply `mark_as_read`. Do not store attachments persistently in SQLite or on disk; prefer incremental transfer for large files when the chosen transport supports it.

This completes the approved initial technical decisions for `RTRM008`; implementation details not yet specified (such as an exact transport framing for large binary transfers) remain subject to later validation.

**Related requirements:** RFRM017–RFRM018, RFRM022–RFRM027, RFRM062; RFGS010–RFGS012; RTCA session and connection rules.

### Conversation reading

#### RTRM009 — Open a conversation from a selected message

The distinct `read_thread` operation must open a complete discoverable conversation from an **explicitly selected message**, rather than changing the behavior of `read_message`:

1. Require the selected message's **folder, `UIDVALIDITY`, and UID** as the entry-point identity. Open the specified mailbox and validate its current `UIDVALIDITY` and the existence of that UID before determining conversation membership.
2. Require the connected IMAP server to advertise **`THREAD=REFS`** for `read_thread` in the initial release. If that capability is unavailable, inform the client that conversation reading is unsupported and make the separate `read_message` operation available; do not overwrite or change the saved `thread_view` preference.
3. Use the identified message only as the **starting point** for discovering its conversation. It is not a permanent conversation root, and discovery must not depend on the continued presence or original folder of any arbitrarily chosen root message.
4. Search for related messages across **all accessible folders of the authenticated account**, including `Sent`, `Archive`, and other available mailboxes, not just the selected folder.
5. The scope of `read_thread` is **independent of `thread_multifolder`**. That preference controls grouped-list counters and group-wide actions, not the scope of conversation reading.
6. The discoverable set of related physical messages must not depend on which existing member of that conversation was selected as the entry point. The detailed IMAP cross-folder discovery strategy is subject to a separate approved decision.

**On-demand discovery of related messages across folders:**

1. Starting from the validated entry-point message, use UID-targeted `FETCH` to obtain its `Message-ID`, `In-Reply-To`, `References`, and `Date` headers together with `INTERNALDATE`, without retrieving its body or attachments.
2. Search **all accessible folders of the authenticated account** using IMAP `UID SEARCH HEADER` criteria for the known message identifiers in the relevant threading headers (`Message-ID`, `In-Reply-To`, and `References`). Do not treat a shared subject or sender as evidence of a conversation relationship.
3. Treat search results as **candidates**, not confirmed members: retrieve their threading headers and `Date` header together with `INTERNALDATE`, and verify their explicit identifier relationships, since `SEARCH HEADER` may match substrings rather than whole message identifiers. **Batch candidate UID FETCH requests by selected folder and sets of UIDs** rather than issuing one request per message.
4. Expand the set of known identifiers and repeat searches as needed until no new related physical messages can be discovered. Relationships preserved in `References` can connect surviving messages through identifiers of intermediate messages that were moved or deleted.
5. Retain each discovered physical message separately, identified by **folder, `UIDVALIDITY`, and UID**, even when multiple physical copies share the same `Message-ID`. Do not deduplicate physical copies by message identifier.
6. Discover membership using only the necessary message headers; **do not download bodies or attachments** in this phase. Do not persist a local account-wide message or conversation index.
7. Return the related messages actually discovered; do not claim that the absence of additional results proves that no other related messages exist.

**Chronological ordering without an extra date-fetch phase:**

1. During the same `UID FETCH` operations used to discover and validate conversation candidates, request `Date` via `BODY.PEEK[HEADER.FIELDS (MESSAGE-ID IN-REPLY-TO REFERENCES DATE)]` and request `INTERNALDATE` as an additional FETCH item. Each request can target a UID set in its selected mailbox; a single FETCH cannot span multiple selected mailboxes.
2. Retain the retrieved date information alongside each discovered physical message. Do not perform a separate second round of IMAP requests solely to retrieve message dates.
3. After discovery has finished, order **only the discovered related messages**, across all their folders, by the parsed `Date` header in ascending chronological order (oldest first), accounting for time-zone offsets. If `Date` is missing or cannot be parsed, fall back to `INTERNALDATE`.
4. Preserve **all distinct physical copies** of messages in the returned result, even when they have the same `Message-ID`. Each entry retains its source folder, `UIDVALIDITY`, and UID. Do not deduplicate by `Message-ID`.

**Retrieve conversation message contents without duplicating `read_message`:**

1. After identifying and chronologically ordering the related physical messages, reuse the already-approved `RTRM008` rules for each message's metadata, MIME structure, available HTML/plain-text bodies, HTML sanitization, and referenced CID images. `read_thread` must not introduce an independent message-decoding or HTML-security implementation.
2. Group retrievals **by source IMAP folder**, batching `UID FETCH` commands for sets of related UIDs when compatible with the required attributes and MIME sections. Do not issue one server request per message when several messages in the same selected folder can be retrieved together.
3. Retrieve only the message-body MIME sections required for viewing. Do not fetch the full raw message through `BODY.PEEK[]` or proactively download other downloadable attachment contents. Return attachment metadata so individual attachments can be requested separately using `RTRM008`.
4. Include each physical message's own folder, `UIDVALIDITY`, UID, current IMAP flags, and MIME metadata in the returned result, preserving distinct copies even when `Message-ID` values match.
5. Return the discovered messages **in the approved chronological order, oldest first**, independently of their source folders or the selected entry-point message.
6. Sanitize the HTML and apply embedded-CID and externally hosted resource restrictions **separately for each message**. A conversation can contain multiple senders; an authorization applicable to one sender must not implicitly authorize remote content in another sender's message.
7. Body retrieval must not implicitly alter `\Seen`: use the `BODY.PEEK` behavior agreed in `RTRM008`. Automatic read-state changes are handled separately and must affect only the explicitly selected entry-point message under the next approved decision.

**Automatic read state when opening a conversation:**

1. Retrieve all discovered conversation message bodies using `BODY.PEEK`, as already approved for `RTRM008`; do not implicitly alter their IMAP read flags.
2. Identify the **explicitly selected entry-point message** using its folder, `UIDVALIDITY`, and UID; this is the only message eligible for automatic read-state modification.
3. Check the effective `mark_as_read` preference and that message's current `\Seen` flag. If enabled and the selected message is unread, issue `UID STORE +FLAGS.SILENT (\Seen)` **only for its UID** in its original mailbox, preserving every other flag and reusing the safe identity checks already defined in `RTRM008`.
4. If `mark_as_read` is disabled or the selected message is already read, do not change flags. **Never mark other related messages as read** just because `read_thread` returned their contents.
5. Reflect a successful flag change in the result and distribute it through the approved account-session synchronization mechanism.
6. If updating `\Seen` fails, still return the conversation contents already retrieved and report the flag-update failure separately, without claiming a successful read-state change.

This completes the initially approved technical behavior for `RTRM009`; any additional implementation decisions remain subject to separate validation.

**Related requirements:** RFRM017–RFRM021, RFRM053–RFRM055, RFRM062; RTRM008.

### Message flag operations

#### RTRM010 — Explicitly set or clear message flags

**Mark one or multiple messages as read or unread (`\Seen`):**

1. Accept one or multiple message identities, each including **folder, `UIDVALIDITY`, and UID**, and the explicitly requested target read state. This operation must be available independently of the account's automatic `mark_as_read` preference.
2. Group selected messages by mailbox; validate the current `UIDVALIDITY` for each selected mailbox before modifying any UID belonging to it. Respect server permissions for changing message flags.
3. To mark messages as read, issue UID-targeted `UID STORE +FLAGS.SILENT (\Seen)`; to mark them as unread, use `UID STORE -FLAGS.SILENT (\Seen)`.
4. Where multiple messages reside in the same mailbox, batch their UIDs into a UID set instead of sending a separate `STORE` for each message.
5. Modify **only `\Seen`**, preserving all unrelated IMAP flags and custom keywords, including `\Answered`, `\Flagged`, and `\Draft`.
6. Report permission, identity, missing-message, or other IMAP failures without claiming that failed operations succeeded. Notify the other authorized sessions through the existing IMAP synchronization mechanism so affected individual message states, folder unread counters, and conversation unread counters are refreshed.

**Flag or unflag one or multiple messages (`\Flagged`):**

1. Accept one or multiple message identities containing **folder, `UIDVALIDITY`, and UID** and the explicit target flagged state.
2. Group target messages by mailbox and validate the corresponding `UIDVALIDITY` and server flag-modification permissions before applying updates.
3. To flag messages, execute `UID STORE +FLAGS.SILENT (\Flagged)`; to remove the flag, execute `UID STORE -FLAGS.SILENT (\Flagged)`.
4. Batch multiple UIDs belonging to the same mailbox into a single UID set where possible; reuse the flag-update mechanism already established for `\Seen` rather than duplicating its implementation.
5. Modify **only `\Flagged`**. Preserve `\Seen`, `\Answered`, `\Draft`, other IMAP flags, and custom keywords; this explicit operation is independent of the `mark_as_read` preference and must not change read state.
6. Respect IMAP server permissions and report failed or partial operations accurately. Notify other authorized sessions and refresh affected message flags and `thread.flagged` counters through the existing synchronization mechanism.
7. A change to `\Flagged` must not invalidate or alter the cached conversation-membership structure, although affected flag-derived counters or views must be refreshed.

**Mark all unread messages in one folder as read:**

1. Provide an explicit operation for a **single selected mailbox**. Select that IMAP mailbox and verify that the current session has permission to modify the `\Seen` flag.
2. Run `UID SEARCH UNSEEN` against **only that selected mailbox** to obtain the UIDs currently lacking `\Seen`. Do not traverse or include subfolders.
3. If no unread UIDs are returned, complete without issuing a `UID STORE` command.
4. Otherwise, run `UID STORE +FLAGS.SILENT (\Seen)` on the returned UID set, splitting into UID batches if necessary to avoid excessively large commands. Modify only `\Seen`, preserving all other standard and custom flags.
5. Do not retrieve message bodies, attachments, or full message metadata to perform this operation.
6. Report failures, including partial failures across batches, without representing the entire operation as successful. After successful changes, synchronize the affected folder's unread count, individual message read flags, and any affected `thread.unread` conversation counters with other authorized account sessions.
7. Changing read flags does **not** change conversation membership or invalidate the cached `THREAD=REFS` structure, although cached or displayed flag-derived counters must be refreshed.

**Explicitly mark an open conversation as read or unread (independent of `thread_multifolder`):**

1. When the user requests an explicit whole-conversation read/unread action from the conversation opened with `read_thread`, accept the **complete set of related physical-message identities** already returned by `read_thread` (folder, `UIDVALIDITY`, and UID for each message). The scope is **all those discovered messages across the account**, regardless of `thread_multifolder` and `mark_as_read`.
2. Reuse those identities without repeating conversation discovery when they remain valid. Group UIDs by mailbox, validate each mailbox's current `UIDVALIDITY`, and respect the server's permission to modify `\Seen`. Report invalid or missing identities rather than applying changes to unintended messages.
3. **Mark as read:** obtain the currently unread UIDs in each relevant mailbox using `UID SEARCH UNSEEN`, intersect them with the conversation's UID set for that mailbox, and issue `UID STORE +FLAGS.SILENT (\Seen)` only for the matching UIDs.
4. **Mark as unread:** issue `UID STORE -FLAGS.SILENT (\Seen)` for all supplied conversation UIDs in their respective mailboxes.
5. Batch IMAP operations by mailbox and use UID sets (splitting into batches if required). Do not download message bodies or attachments, and do not change flags other than `\Seen`.
6. Propagate successful changes to individual message states, mailbox unread counters, and affected conversation `thread.unread` counters across authorized account sessions. Report partial failures accurately without indicating that an unsuccessful mailbox or message update completed.

**Explicit conversation read/unread actions from a grouped folder listing when `thread_multifolder = false`:**

1. Accept the folder, `UIDVALIDITY`, and UID of the conversation's representative message supplied by the grouped folder listing.
2. Validate that representative's mailbox identity and determine its related physical messages **only within the selected folder**, reusing the cached `THREAD=REFS` structure when valid. Do not include messages from other folders.
3. **Mark as read:** identify which of the conversation's UIDs are still unread (using `UID SEARCH UNSEEN` in the selected mailbox) and issue `UID STORE +FLAGS.SILENT (\Seen)` only for those UIDs.
4. **Mark as unread:** issue `UID STORE -FLAGS.SILENT (\Seen)` for all conversation UIDs found in the selected mailbox.
5. Use UID sets and batched commands when appropriate. Do not retrieve message bodies or attachments, and preserve every flag other than `\Seen`.
6. Synchronize affected individual flags, mailbox unread counts, and `thread.unread` counters. Respect IMAP permissions, report failed or partial operations accurately, and keep cached conversation membership unchanged.
7. This **grouped-list action** uses the effective `thread_multifolder` scope; it is distinct from the already approved read/unread operation on an **open conversation**, which always covers all its discovered related messages.

**Explicit conversation read/unread actions from a grouped folder listing when `thread_multifolder = true`:**

1. Accept the **folder, `UIDVALIDITY`, and UID** of the representative message supplied by the grouped folder listing and validate its identity.
2. Resolve the related physical messages across **all accessible folders of the account**, reusing the approved `RTRM009` cross-folder discovery mechanism **without retrieving bodies or attachments**.
3. Reuse related-message identities already available in the current session cache where they remain valid, avoiding redundant discovery when possible.
4. **Mark as read:** identify which discovered related messages remain unread and apply `UID STORE +FLAGS.SILENT (\Seen)` to only those messages in their respective mailboxes.
5. **Mark as unread:** apply `UID STORE -FLAGS.SILENT (\Seen)` to all discovered related messages in their respective mailboxes.
6. Group operations by mailbox, use UID sets and batches as appropriate, and validate each selected mailbox's `UIDVALIDITY` and IMAP permissions. Preserve all other flags.
7. Synchronize the resulting individual message flags, affected folder unread counters, and `thread.unread` counters across other authorized account sessions. Report permission failures and partial updates accurately.
8. This grouped-list action respects `thread_multifolder`; **open-conversation** read/unread actions remain account-wide regardless of that preference.

**Still pending within RTRM010:** any additional flag synchronization details.

**Related requirements:** RFRM028–RFRM032, RFRM058, RFRM062; RTRM008–RTRM009; RTSN.

## Scope boundaries and follow-up documentation

- **Quota information:** `RTRM002` implements only on-demand `STORAGE` usage and limit, beginning with `GETQUOTAROOT INBOX`. A complete per-mailbox quota-root inventory and other quota resources are outside the initial scope.
- **Folder modifications and listings:** `RTRM003` covers folder creation, `RTRM004` covers renaming and movement, `RTRM005` covers deletion, and `RTRM006` covers individual-message listings. `RTRM007` covers grouped-conversation listings, while `RTRM008` begins the individual-message reading operation; remaining message operations will follow.
- **Server authority:** actual mailbox inventory, status values, attributes, and access restrictions come from the IMAP server.
