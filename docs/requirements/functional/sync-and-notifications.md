# Sync and notifications

**Group:** `RFSN`

**Numbering:** `RFSN001`, `RFSN002`, `RFSN003`…

**Implementation details:** [Sync and notifications — technical requirements](../technical/sync-and-notifications.md).

## Scope

This group specifies **what Postela must do** to detect mail changes, keep a client application's state up to date, and recover from interrupted connections. It does not prescribe a frontend framework or a particular implementation protocol.

**Postela does not store email messages.** Messages remain on the mail server and are retrieved when needed.

Reading, modifying, and sending messages, as well as initial authentication and general session management, are covered by their respective requirements groups.

## Functional requirements

### Change monitoring

#### RFSN001 — Continuous mail monitoring

While an account session is active, Postela must detect changes to the account's mail without requiring the user to refresh manually.

#### RFSN002 — Monitoring all personal folders

Postela must monitor changes in all personal folders of an account, including folders that the user is not currently viewing.

#### RFSN003 — Detecting message arrivals and removals

Postela must detect when messages appear in or disappear from a folder, including changes caused by receiving, deleting, or moving messages between folders.

#### RFSN004 — Detecting message flag changes

Postela must detect changes to message flags, such as read and unread status, regardless of which folder contains the message.

#### RFSN005 — Detecting folder changes

Postela must detect folder creation, deletion, renaming, and subscription changes.

### Client state updates

#### RFSN006 — Updating folder counters

When a folder's contents or state changes, Postela must update the affected counters, including the total number of messages and the number of unread messages.

#### RFSN007 — Refreshing the visible message page

When the contents or message state of the folder currently being viewed change, Postela must reload **only the page of messages currently visible to the user**, preserving the selected pagination and sort order, and replace the displayed list with the refreshed result.

For a folder that is not currently being viewed, Postela must update its counters without refreshing its message list.

#### RFSN008 — Updating the folder tree

The folder tree presented to the user must reflect folder creation, deletion, renaming, and subscription changes without a manual refresh.

### Real-time communication

#### RFSN009 — Bidirectional communication

Postela must allow a client application to submit requests, receive responses, and obtain real-time change notifications throughout an active session.

#### RFSN010 — Notification isolation

Changes to an account must be delivered only to sessions authorized to access that account. A session must never receive notifications for an account it is not authorized to access.

### Disconnections and recovery

#### RFSN011 — Detecting loss of mail-server connectivity

Postela must detect when the connection used to monitor the mail server is no longer available.

#### RFSN012 — Reconnecting to the mail server

After an unexpected interruption of the mail-monitoring connection, Postela must make **up to three reconnection attempts** before considering the mail server unavailable.

#### RFSN013 — Resuming mail monitoring

After successfully reconnecting to the mail server, Postela must resume monitoring changes in all personal folders.

#### RFSN014 — Mail-server reconnection failure

If all three mail-server reconnection attempts fail, Postela must end the affected session and inform the user that it cannot connect to the mail server.

#### RFSN015 — Reconnecting a client to the backend

If a client application's connection to the Postela backend is interrupted unexpectedly, the client must make **up to three reconnection attempts** before considering the backend unavailable.

#### RFSN016 — Handling intentional connection closure

When Postela intentionally closes a connection and communicates the reason, the client must act on that reason and must not attempt to reconnect if the session has been definitively terminated.

If the connection closes without a received indication of intentional termination, the client must treat the interruption as unexpected.

#### RFSN017 — Backend reconnection failure

If all three attempts to reconnect to the backend fail, the client must log the user out, end its local session, and explain that it cannot connect to the backend.

#### RFSN018 — Restoring client state after reconnection

After connectivity to the mail server or backend is successfully restored, Postela must rebuild the client application's state **as it would after sign-in**, without requesting credentials again if the session remains valid.

Recovery must include the folder tree, folder counters, and the currently displayed message page. It must not retrieve every message in the account.
