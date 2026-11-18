# Reading and management

**Group:** `RTRM`

**Numbering:** `RTRM001`, `RTRM002`, `RTRM003`…

**Related requirements:** [Functional reading and management](../functional/reading-and-management.md) (`RFRM`), [Technical general settings](./general-settings.md) (`RTGS`), [Core architecture](./core-architecture.md) (`RTCA`), and [Sync and notifications](./sync-and-notifications.md) (`RTSN`).

## Scope

This document specifies the technical implementation of mailbox reading and management through IMAP, following the functional requirements in `RFRM`.

The requirements are introduced incrementally as their behavior is agreed. `RTRM001` covers folder hierarchy retrieval and status counters; `RTRM002` covers on-demand account storage usage and limit. Folder mutations, message listings, threading, searches, and other operations will be specified in subsequent requirements.

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

## Scope boundaries and follow-up documentation

- **Quota information:** `RTRM002` implements only on-demand `STORAGE` usage and limit, beginning with `GETQUOTAROOT INBOX`. A complete per-mailbox quota-root inventory and other quota resources are outside the initial scope.
- **Folder modifications and message operations:** not specified by `RTRM001`; their implementation will be added incrementally.
- **Server authority:** actual mailbox inventory, status values, attributes, and access restrictions come from the IMAP server.
