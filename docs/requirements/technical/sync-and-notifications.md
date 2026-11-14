# Sync and notifications

**Group:** `RTSN`

**Numbering:** `RTSN001`, `RTSN002`, `RTSN003`…

**Related functional requirements:** [Sync and notifications](../functional/sync-and-notifications.md).

## Scope

This document specifies the technical mechanisms used by the **Postela backend** for mail monitoring, real-time communication, and connection recovery. User-visible behavior is defined separately in the `RFSN` functional requirements.

The backend is implemented in Go. It exposes an interface usable by **clients built with any frontend framework or technology**. No frontend framework is part of the backend contract.

**Postela does not store email messages.** Messages remain on the IMAP server and are retrieved on demand. Application-specific configuration and user data are covered by other requirements groups.

## Technical requirements

### IMAP monitoring

#### RTSN001 — Persistent monitoring connection

The backend must maintain **one persistent IMAP monitoring connection per monitored account context**, rather than a separate persistent connection for every folder.

This connection must be dedicated to receiving events. Ordinary message reads and modifications must use separate, short-lived IMAP connections established on demand. Monitoring must not require selecting each folder with `SELECT`.

**Related functional requirements:** RFSN001–RFSN004.

#### RTSN002 — IMAP transport and authentication

The monitoring connection must use authenticated IMAP over STARTTLS, with server-certificate validation. Monitoring must not be marked operational until authentication has succeeded.

Initial user authentication and credential handling will be specified in the authentication and session requirements.

**Related functional requirements:** RFSN001, RFSN011.

#### RTSN003 — Required IMAP server capabilities and NOTIFY subscription

**The mail server must support and advertise the IMAP `NOTIFY`, `CONDSTORE`, and `ENABLE` capabilities to authenticated clients.** It must support the `PERSONAL` notification selector and the required events for personal folders, including flag changes in non-selected folders when `CONDSTORE` is enabled.

After authentication, the backend must issue `ENABLE CONDSTORE` and establish a `NOTIFY SET` subscription with the `PERSONAL` selector, requesting:

- `MessageNew`
- `MessageExpunge`
- `FlagChange`
- `MailboxName`
- `SubscriptionChange`

The backend must verify that enabling `CONDSTORE` and establishing the subscription both succeed. Monitoring is not operational if these prerequisites are missing or either operation fails.

The subscription must cover personal folders without issuing `SELECT` for each folder. **Neither `IDLE` nor per-message `MODSEQ` tracking or `CHANGEDSINCE`-based incremental synchronization is required or used for this monitoring design.**

**Related functional requirements:** RFSN001–RFSN004, RFSN011.

#### RTSN004 — Asynchronous IMAP response handling

The backend's IMAP client must continuously process unsolicited server responses, even when no IMAP command is outstanding.

Message and flag changes are received through `STATUS` responses; folder structure and subscription changes are received through `LIST` responses. Event handlers must not block the connection's response-reading loop.

**Related functional requirements:** RFSN001–RFSN005.

#### RTSN005 — Handling partial STATUS responses

An unsolicited `STATUS` response may contain only the attributes affected by a change. The backend must inspect which attributes are actually present; absent attributes must **not** be interpreted as zero values or changes.

Relevant attributes include `MESSAGES`, `UNSEEN`, `UIDNEXT`, and `HIGHESTMODSEQ` when supplied. `HIGHESTMODSEQ` may indicate flag changes but must not be used to maintain an incremental local message store.

**Related functional requirements:** RFSN002–RFSN004, RFSN006.

#### RTSN006 — Handling LIST and folder changes

The backend must interpret unsolicited `LIST` responses for `MailboxName` and `SubscriptionChange` events.

It must recognize `OLDNAME` for renames, `\\NonExistent` for deletions, and `\\Subscribed` for subscription state.

Multiple `LIST` responses may describe a single operation. A response without these attributes must not, on its own, be treated as proof that a folder has just been created.

**Related functional requirements:** RFSN005, RFSN008.

### Data updates and client communication

#### RTSN007 — WebSocket connection per session

The Postela backend must support **one bidirectional WebSocket connection per active client session**.

That channel must carry requests, responses, and asynchronous notifications for all modules. The protocol must not require additional WebSocket connections for each folder or feature.

The client may be implemented using any technology capable of speaking the defined protocol.

**Related functional requirements:** RFSN009.

#### RTSN008 — Notification routing

The backend must associate each detected IMAP change with its account and route notifications only to sessions authorized for that account.

The notification protocol must distinguish changes affecting messages, flags, or folder structure and must identify the affected folder.

The precise WebSocket message format remains an implementation decision.

**Related functional requirements:** RFSN006–RFSN010.

#### RTSN009 — Reloading the visible message page

When the client requests a refresh of the open folder following a change, the backend must retrieve **only the currently visible page of messages**, using the requested pagination and sort order, through an on-demand IMAP connection.

The client replaces its displayed message list with the response. The backend must not calculate differences between successive message lists, fetch unrelated pages, or identify individual changed message UIDs solely to refresh the list.

For folders that are not open, only the relevant counters and folder state need updating.

**Related functional requirements:** RFSN006, RFSN007.

#### RTSN010 — No message storage

The backend must retrieve email messages directly from the IMAP server when requested and deliver the requested data to the authorized client. **It must not store email messages in SQLite or in any other persistent local message repository.**

Message data may be processed in memory as needed to fulfill a request; this does not authorize local storage of a message archive.

**Related functional requirements:** RFSN007, RFSN018.

### Disconnections and recovery

#### RTSN011 — Detecting IMAP disconnections

The backend must detect when the persistent IMAP monitoring connection terminates or fails and mark it unavailable. Detection must not depend on the user issuing a new operation.

The exact method and timeouts for detecting inactive connections remain to be decided.

**Related functional requirements:** RFSN011.

#### RTSN012 — IMAP reconnection and subscription restoration

After an unexpected IMAP disconnection, the backend must make **up to three reconnection attempts**.

For each successful network reconnection, the backend must authenticate again, enable `CONDSTORE`, and reestablish the `NOTIFY PERSONAL` subscription with the events defined in RTSN003.

Monitoring may be considered restored only after these steps succeed. Retry intervals and timeouts remain to be determined.

**Related functional requirements:** RFSN012, RFSN013.

#### RTSN013 — Final IMAP reconnection failure

After all three IMAP reconnection attempts fail, the backend must send a reason identifying the mail-server connectivity failure, intentionally close the affected client's WebSocket connection, and terminate the affected session.

The close reason must allow the client to recognize definitive session termination without starting another three WebSocket reconnection attempts for the same failure.

**Related functional requirements:** RFSN014, RFSN016.

#### RTSN014 — WebSocket reconnection

The client application is responsible for reconnecting its WebSocket after an unexpected interruption, with **a maximum of three attempts**.

If the connection is restored and the session remains valid, the client may resume without asking the user to enter credentials again. The backend's session-retention behavior while the WebSocket is unavailable will be specified in the session requirements.

Retry intervals and timeouts remain to be determined.

**Related functional requirements:** RFSN015.

#### RTSN015 — Identifying intentional WebSocket closure

Intentional WebSocket termination must convey a reason identifiable by the client, using an appropriate WebSocket close code or an application message delivered before closing.

A definitive termination reason must suppress automatic reconnection. If the client does not receive a reliable indication of intentional termination, it must apply the unexpected-disconnection retry policy.

Exact close codes and message formats remain to be decided.

**Related functional requirements:** RFSN014, RFSN016.

#### RTSN016 — Final WebSocket reconnection failure

After three unsuccessful WebSocket reconnection attempts, the client must clear its local session, log the user out, and display an error identifying the backend as unreachable.

**Related functional requirements:** RFSN017.

#### RTSN017 — Restoring state after reconnection

After IMAP or WebSocket connectivity is successfully restored, Postela must run the initial client-state retrieval flow **without requesting credentials again while the session remains valid**.

The recovery flow must reload the folder tree, folder counters, and the currently visible message page. It must not download all messages or replay an incremental history of events missed during the disconnection.

**Related functional requirements:** RFSN018.

## Compatibility verified

The core monitoring mechanism was tested against **Dovecot 2.3.21.1** using **Go 1.24.4** and `github.com/kiliant/go-imap` **v1.3.1** as a candidate library.

The tests confirmed STARTTLS authentication, `ENABLE CONDSTORE`, `NOTIFY PERSONAL`, and asynchronous `STATUS` and `LIST` notifications for new messages, read/unread changes, moves between folders, folder creation, subscriptions, renames with `OLDNAME`, and deletions with `\\NonExistent`.

These tests establish compatibility for the tested environment. **They do not constitute a final decision to adopt the library**, nor do they verify resilience during prolonged network outages or automatic reconnections.

## Open technical decisions

Retry delays and timeouts, idle-connection failure detection, exact WebSocket message formats and close codes, and how long the backend retains a session after WebSocket loss remain to be specified. These choices must preserve the functional behavior defined in the related `RFSN` requirements.
