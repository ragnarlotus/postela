# Reading and management

**Group:** `RTRM`

**Numbering:** `RTRM001`, `RTRM002`, `RTRM003`…

**Related requirements:** [Functional reading and management](../functional/reading-and-management.md) (`RFRM`), [Technical general settings](./general-settings.md) (`RTGS`), [Core architecture](./core-architecture.md) (`RTCA`), and [Sync and notifications](./sync-and-notifications.md) (`RTSN`).

## Scope

This document specifies the technical implementation of mailbox reading and management through IMAP, following the functional requirements in `RFRM`.

The requirements are introduced incrementally as their behavior is agreed. `RTRM001` covers folder hierarchy retrieval and status counters; `RTRM002` covers on-demand account storage usage and limit; `RTRM003` covers creation of individual mailboxes and subfolders. Other folder mutations, message listings, threading, searches, and remaining operations will be specified in subsequent requirements.

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

## Scope boundaries and follow-up documentation

- **Quota information:** `RTRM002` implements only on-demand `STORAGE` usage and limit, beginning with `GETQUOTAROOT INBOX`. A complete per-mailbox quota-root inventory and other quota resources are outside the initial scope.
- **Folder modifications and message operations:** `RTRM003` specifies individual folder creation. Folder rename, movement, deletion, and message operations will be added incrementally.
- **Server authority:** actual mailbox inventory, status values, attributes, and access restrictions come from the IMAP server.
