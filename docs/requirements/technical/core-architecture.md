# Core architecture

**Group:** `RTCA`

**Numbering:** `RTCA001`, `RTCA002`, `RTCA003`…

**Related technical requirements:** [Sync and notifications](./sync-and-notifications.md). Other technical requirement groups, including data storage, will be specified separately.

## Scope

This document defines the foundational technical requirements of the Postela backend: application structure, configuration, authentication, session management, security, client communication, resource limits, logging, startup, and shutdown.

Requirements describe the **agreed architecture and observable technical contracts**, not a mandatory class hierarchy or an implementation for each individual operation.

Postela is a backend service written in Go. Client applications may use any framework or technology capable of implementing its protocol. The backend must not depend on any particular frontend implementation.

Email messages remain on the mail server. **Postela does not persist email messages locally.** Message retrieval and IMAP notification handling are covered in their respective requirement groups.

## Technical requirements

### Application architecture

#### RTCA001 — Go backend

The Postela backend must be implemented in Go. It must provide the application services and communication interfaces required by compatible clients without depending on a particular frontend framework.

#### RTCA002 — Single-instance execution

The initial release must run as **one backend process or service instance** handling multiple accounts and concurrent sessions.

Distributed session coordination, horizontal scaling, and dependencies such as Redis for coordinating backend instances are outside the initial scope.

#### RTCA003 — Cohesive and maintainable code organization

The implementation must organize code into cohesive, clearly named responsibilities with explicit boundaries.

SOLID and related design principles should guide the implementation **pragmatically, not dogmatically**. Modules, packages, types, and interfaces should be introduced where they materially improve clarity, reuse, maintainability, or separation of concerns. The design must avoid unnecessary wrappers, trivial abstractions, excessive fragmentation, and indirect call chains.

The organization should follow the principle demonstrated by Proxmox Forge: keep closely related logic together and separate substantial responsibilities, rather than creating a module for every operation.

### Configuration

#### RTCA004 — TOML configuration files

Configuration must use TOML rather than indentation-sensitive formats.

The repository must provide a version-controlled `config.default.toml` containing the default configuration. Deployments may provide a `config.toml` with **only the settings that differ from the defaults**.

`config.toml` must be excluded from version control and must not be automatically overwritten by application updates.

#### RTCA005 — Layered configuration precedence

Effective configuration must be computed in this order, from lowest to highest precedence:

1. `config.default.toml`.
2. `config.toml`.
3. Supported environment-variable overrides.

Overrides must apply at the individual configuration-key level. Replacing one key inside a TOML section must not discard unrelated default keys from the same section.

New defaults introduced in an update must remain available to installations whose override file does not mention them.

#### RTCA006 — Environment-variable overrides

Supported settings must be overridable through environment variables, particularly when deployment secrets are involved.

The mapping between configuration keys and environment variables must be documented. Environment variables must not require modifying either TOML file.

#### RTCA007 — Configuration validation and unknown keys

The backend must validate the effective configuration before accepting client connections.

Invalid values for recognized settings, missing mandatory values, and incompatible settings that prevent correct operation must produce an actionable startup error.

**Unknown or obsolete keys in `config.toml` must be ignored and reported at `WARN` level**, identifying the affected key. They must not, by themselves, prevent startup.

#### RTCA008 — Configuration changes and restarts

Dynamic configuration reload is not required for the initial release. Configuration changes take effect after restarting the backend.

A restart also invalidates all in-memory sessions as specified in RTCA021.

#### RTCA009 — Centralized mail-server configuration

IMAP, SMTP, and ManageSieve server settings must be centrally administered by the backend.

Server settings must be configurable **per email domain**, with **per-account exceptions** overriding the relevant domain settings.

A client must not be able to select an arbitrary mail server by supplying its own server address during login or subsequent operations.

### Account identity and authentication

#### RTCA010 — Email address as account identifier

An email address must uniquely identify a Postela mail account. A given email address must resolve to one configured mailbox through the backend's centrally managed configuration.

The IMAP authentication username may differ from the email address; it must not replace the email address as the Postela account identifier.

#### RTCA011 — Independent IMAP authentication per login

Every new user session must authenticate against the configured IMAP server using its own credentials, **even when another session for the same email account is already active**.

The existence of another authenticated session or a shared monitoring connection must not bypass authentication for a new session.

#### RTCA012 — In-memory credential handling

Credentials needed for authenticated sessions, mail operations, and restoration of persistent IMAP monitoring must be held **in backend memory only**.

Passwords and other mail-server credentials must never be written to SQLite, configuration files, application log files, or any other persistent store by Postela.

Credentials associated with an invalidated session must be released when no longer needed. Credential use by a shared monitoring connection is governed by RTCA024.

#### RTCA013 — Opaque session tokens

After successful authentication, the backend must create an independent session identified by a **cryptographically secure, unpredictable, opaque token**.

Each session must have its own token, even when multiple sessions belong to the same email account.

The backend must manage token validation, expiration, and immediate revocation. JWT is not required for this architecture.

#### RTCA014 — Session-token transport

For browser clients, authentication must use cookies with appropriate `HttpOnly`, `Secure`, and `SameSite` attributes, so client-side JavaScript does not need direct access to the session token.

Native and other non-browser clients must be able to present the same kind of opaque session token using `Authorization: Bearer`.

Session tokens must not be placed in URLs or stored in browser `localStorage` or `sessionStorage`.

This does not introduce a second token type for different clients: cookie and Bearer are alternative ways to present the session credential.

#### RTCA015 — Authentication flow and WebSocket establishment

The initial login and session creation must use HTTPS.

After authentication, the client establishes its **one authenticated WebSocket connection for that session**. The session token is validated during the WebSocket handshake and need not be repeated in each subsequent WebSocket message.

The backend must continue enforcing session validity and authorization for operations made over an established WebSocket.

The client may optionally provide a stable application identifier (`client_id`) during HTTPS login. If omitted, the session's `client_id` must be `null`; mail functions and account-wide preferences must remain available without it.

When provided, `client_id` must be a nonempty, case-sensitive ASCII string of **at most 128 characters**, using only letters, digits, periods (`.`), hyphens (`-`), and underscores (`_`). No reverse-domain or other naming convention may be required. An invalid supplied identifier must be rejected as an invalid login request.

The backend must bind the identifier to the authenticated session and keep it **immutable for the lifetime of that session**, including WebSocket reconnections. The WebSocket must inherit it from the session, without relying on an identifier declared in each message. If a client-preference operation supplies an identifier, the backend must reject it unless it matches the session-bound value.

The identifier only distinguishes client applications; it is **not an authentication credential** and must not be treated as an authorization boundary between applications of the same authenticated account. Generic application preference storage is defined in [Functional data storage](../functional/data-storage.md) and [Technical data storage](./data-storage.md).

#### RTCA016 — Browser-origin and request protection

The backend must validate browser WebSocket `Origin` values against explicitly authorized origins and must apply suitable protections against cross-site requests in browser-facing HTTP endpoints, including login.

The design must not rely on TLS or `SameSite` alone as a substitute for origin checks.

An additional independent CSRF token or per-request token rotation is **not required for the initially agreed login-plus-WebSocket flow**. Any future authenticated HTTP operations must receive an appropriate security review.

#### RTCA017 — Login-abuse protection

Authentication attempts must be rate-limited using configurable thresholds for both the account and the client IP address.

Exceeding the limits must cause a configurable temporary restriction. Authentication failures must not disclose whether the account exists or the password is incorrect.

Failed authentication attempts and restrictions must be recorded at `WARN` level without exposing credentials.

#### RTCA018 — Trusted client IP identification

When deployed behind a reverse proxy, the backend must accept forwarded client-address information only from proxies explicitly configured as trusted.

Client-supplied forwarding headers from untrusted sources must not be accepted as authoritative, including for login-abuse rate limiting.

### Session lifecycle and limits

#### RTCA019 — Session independent of WebSocket lifetime

The authenticated session must have a lifecycle independent of its WebSocket connection.

An unexpected WebSocket disconnection must not immediately invalidate an otherwise valid session. The client must be able to reconnect using that same session without resending the mail-server password.

#### RTCA020 — Disconnected-session expiration

A session with an active, usable client connection may remain valid while the connection and activity continue.

When its WebSocket disconnects, the backend must retain the session for a **configurable reconnection period**. If the client does not reconnect before the period expires, the backend must invalidate the session and release its resources.

A valid reconnection must not create or count as a second session.

#### RTCA021 — Session invalidation and backend restart

Explicit logout must immediately invalidate **only the session being logged out**, including its token and WebSocket, without affecting other sessions of the same account.

The backend must not persist sessions or session tokens for restoration after a process restart. **Restarting the backend invalidates all existing sessions**, and clients must authenticate again.

A known invalid or expired session must receive a definitive authentication/session error. The client must not exhaust the three network-reconnection attempts when authentication is already known to be impossible.

#### RTCA022 — Global and per-account session limits

The backend must enforce configurable limits on:

- Total simultaneously valid sessions: `max_sessions`.
- Simultaneously valid sessions per email account: `max_sessions_per_account`.
- An optional per-account override of the general per-account limit.

Valid but temporarily disconnected sessions must count toward these limits until they reconnect or expire. Requests to create sessions beyond a limit must be rejected with a specific error without terminating existing sessions.

These session limits replace the previously considered independent global WebSocket-connection limit; reconnection of an existing session must not consume an additional session slot.

### Shared mail monitoring

#### RTCA023 — Shared persistent IMAP monitoring by account

All valid sessions for the same email address must share **one persistent IMAP monitoring connection for that account**, not one connection per session.

The backend must distribute monitoring events only to sessions authorized for the corresponding account. Each client session retains its own WebSocket.

Connection establishment and the `NOTIFY PERSONAL` / `CONDSTORE` implementation are specified in [RTSN001–RTSN006](./sync-and-notifications.md).

#### RTCA024 — Shared-monitoring credential ownership

Each session must be independently authenticated even when its account already has a monitoring connection.

If the session whose credentials were used for shared monitoring ends, other valid sessions of the same account must remain operational. The backend must be able to continue or reestablish monitoring using credentials held for another valid session of that account.

The backend must not retain credentials from an invalidated session solely to keep monitoring alive when another valid session can supply them.

#### RTCA025 — Monitoring-connection lifetime

The shared monitoring connection must be established when the first session requiring supervision becomes active.

It must remain available while at least one valid session still requires it, including sessions retained during the agreed WebSocket reconnection period.

When the last valid session expires or logs out, the backend must close the account's persistent monitoring connection and release credentials and resources associated with that account.

Final monitoring failures and the affected-session recovery policy are specified in [RTSN011–RTSN017](./sync-and-notifications.md).

### Client communication protocol

#### RTCA026 — Framework-independent WebSocket protocol

The backend must expose a **bidirectional JSON protocol over WebSocket**. A session must use a single WebSocket for application requests, responses, and asynchronous notifications across modules.

No frontend framework, including Vue, React, or Angular, may be assumed by the backend protocol.

#### RTCA027 — Request, response, and notification envelopes

The WebSocket protocol must support these message categories:

- **Request:** an operation name and its parameters, without requiring a request identifier at this stage.
- **Response:** the operation's result or error, without mandating a corresponding request identifier.
- **Notification:** an event identifier/type and associated event data, without requiring a preceding request.

**Do not require `request_id` or another universal request identifier in the initial protocol solely to correlate requests and responses.** This does not prohibit identifiers if a concrete operation or the finalized transport protocol establishes a justified need.

The exact JSON property names, message schemas, and mechanism for associating responses with outstanding independent queries and queued operations remain **undecided**. The previously approved query-supersession and side-effect-operation queuing behavior in `RTCA030`–`RTCA033` remains in force; do not infer a correlation mechanism from those concurrency rules.

#### RTCA028 — Protocol version negotiation

The WebSocket protocol must be semantically versioned (`MAJOR.MINOR.PATCH`).

A client must communicate its supported protocol version when connecting. The backend must verify compatibility before accepting application operations.

Incompatible versions must produce a specific error followed by controlled connection closure, **without automatic reconnection attempts**. Protocol incompatibility alone must not invalidate an otherwise valid user session.

#### RTCA029 — Consistent errors and session failures

Operation failures must use stable, language-independent error codes, with enough detail for the client to select an appropriate user-facing message.

**Distinguish an operation failure from a partially completed operation across all modules:**

- **Operation failure that prevents completion:** return an identifiable error in the response associated with the originating client request, using the response category defined in `RTCA027`.
- **Operation completed with partial results or recoverable failures:** return the available successful result in the response and send a **separate warning notification** associated with the request that triggered the operation. The client may use this notification to display a toast explaining the partial outcome. Do not embed that warning in the successful result payload or incorrectly report a complete success.

Backend-originated warning notifications must not, solely because they were sent, terminate an active advanced search or discard its session cache. Any independent invalidation resulting from actual relevant IMAP state changes remains subject to the existing search-cache rules.

**Common JSON error object for failed operations:**

1. Represent an operation error as a JSON object with a required stable, language-independent string `code`, a required technical diagnostic string `message`, and optional structured `details` for operation-specific context.
2. For example: `{ "code": "folder-not-found", "message": "One or more requested folders do not exist", "details": { "folders": ["Archive/2025"] } }`.
3. Place this object in the error-bearing response associated with the originating request, under the generic WebSocket response envelope to be finalized in `RTCA027`.
4. The client uses `code` to select a localized user-facing message. `message` is diagnostic information rather than prelocalized display text; `details` provides context without requiring a separate error-object schema for every operation.

The **concrete warning-notification schema, request-correlation fields for warnings, and warning-repeat policy** remain subject to separate decisions. The surrounding WebSocket envelope schema also remains under `RTCA027`.

An ordinary operation failure must not terminate other requests, the authenticated session, or its WebSocket.

An invalid or expired session must produce a recognizable session error and definitive disconnection, requiring new authentication. The client is responsible for presenting localized messages.

#### RTCA030 — Query supersession and queued operations per session

Separate **replaceable read-only queries** from **operations with side effects**. The decision applies within an individual authenticated client session; a request from one session must not cancel or discard work initiated by another session of the same account.

- **Replaceable queries in the same client flow:** the latest frontend request supersedes earlier unfinished queries of that flow. Discard superseded queued query work immediately, request cancellation of work already running when technically possible, and never return a superseded query result to the frontend. A non-cancellable in-progress query may finish internally, but its obsolete result is discarded. Start the latest query as soon as resources permit, without waiting for obsolete work that can be cancelled.
- **Side-effecting operations:** enqueue them in arrival order for that session. Do not cancel, replace, or silently discard an accepted operation because another frontend request arrives; preserve each operation's outcome and report it according to the established success, failure, and uncertain-result rules. This includes nominally read-oriented actions if they have side effects.
- **Independent queries or work streams:** do not invalidate one another merely because they originate in the same session. Backend-initiated notifications and account monitoring are also independent of frontend query supersession.

Continue enforcing configurable resource-concurrency limits and timeouts, with separate session state and coordinated access to shared backend resources and WebSocket writes. The precise classification of operations by cancelability and their flow boundaries must be made explicit in the operation/protocol contract before implementation; this requirement does not assign every request to a single global replaceable stream.

#### RTCA031 — Automatic cancellation of superseded queries

The initial protocol must **not require a separate client-initiated cancellation command** or a cancel request keyed by a request identifier. Instead, receipt of a newer replaceable query in the same session flow automatically supersedes earlier unfinished queries of that flow as established in `RTCA030`.

The backend must remove obsolete queued queries, request cancellation of obsolete running work where feasible, and suppress any obsolete result even if the underlying work cannot be interrupted. Bound remaining work using resource and operation timeouts. **Never use this automatic cancellation to abort or suppress an accepted side-effecting operation.** An explicit cancellation protocol can be reconsidered only if a concrete later need justifies it.

### Operation concurrency and backpressure

#### RTCA032 — Per-session IMAP operation concurrency

Each session must have a configurable maximum number of **simultaneous on-demand IMAP connections used for mail operations**.

These connections are separate from the shared persistent account-monitoring connection, which is excluded from this limit.

An operation must release its IMAP connection when it completes or times out.

#### RTCA033 — Bounded per-session queue for side-effecting operations

Maintain a **FIFO queue per authenticated session for accepted side-effecting operations**, preserving their arrival order. A later frontend request must not supersede, reorder, or discard queued operations merely because they are older. Execute them in queue order subject to available resources and the established safe completion/uncertain-outcome rules.

Keep queue capacity and maximum waiting time administrator-configurable. Reject an operation with an identifiable capacity/timeout error if it cannot be accepted or exceeds its permitted wait; do not silently drop an accepted operation in favor of a newer one. The previously approved WebSocket-disconnection handling in `RTCA035` remains in force.

**Replaceable read-only queries must not accumulate in this FIFO operation queue.** When resource limits prevent immediate execution, only the latest still-relevant query for the same session flow may remain pending; superseded pending queries are removed, and running ones are cancelled where feasible under `RTCA030`–`RTCA031`. Independent queries remain subject to resource limits without superseding one another.

No session's queued work may directly block or exhaust another session's independent queue.

#### RTCA034 — Bounded outbound WebSocket queues

Each connected session must have its own **bounded outbound WebSocket queue**, with configurable capacity.

A slow or unresponsive client must not block IMAP notification processing or message delivery to other sessions.

When its outbound queue reaches the defined limit, only the affected WebSocket must be closed. Its authenticated session must remain eligible for reconnection until the configured expiration time.

The client must recover current state after reconnecting; the backend does not need to persist and replay missed notifications.

#### RTCA035 — Outstanding work after WebSocket loss

If a WebSocket disconnects unexpectedly:

- IMAP operations already executing may finish or run until their timeouts.
- Queued operations for that disconnected client must be discarded.
- The backend must not automatically replay outstanding requests when the client reconnects.
- Operations with uncertain side effects, especially sending a message, **must not be automatically repeated**, to avoid duplicate actions.

The restored client must obtain current state as specified in [RTSN017](./sync-and-notifications.md).

### Logging

#### RTCA036 — Structured logs and configurable levels

The backend must produce **structured JSON logs**, supporting the levels `DEBUG`, `INFO`, `WARN`, and `ERROR`.

The configured logging level must control which entries are emitted, without changing application code.

**Logging failures is a cross-cutting requirement for every backend subsystem**, including IMAP, SMTP, ManageSieve, SQLite, authentication, session management, WebSocket communication, configuration, and internal background operations. Failures must not be silently ignored merely because the operation also reports an error to a client.

Log failures according to their significance:

- **`ERROR`:** operations that ultimately fail, unrecoverable internal errors, database failures, or other problems that prevent the requested work from completing correctly.
- **`WARN`:** unexpected or recoverable conditions requiring attention, including transient failures for which recovery is attempted and relevant rejected or invalid requests.
- **`INFO` / `DEBUG`:** ordinary activity and diagnostic details, including individual retry steps where useful; these must not replace the final error log when an operation ultimately fails.

Each failure log must provide enough structured context to identify the affected subsystem, operation, cause, and relevant account/session/request identifiers where available, following the privacy restrictions of RTCA037. The same failure should not be redundantly emitted at every call layer when one contextualized log entry suffices.

As with all logs, emission remains subject to the configured minimum logging level.

#### RTCA037 — Traceability and sensitive-data protection

Logs must include relevant account, session, and request identifiers when needed to diagnose an operation, while avoiding unnecessary disclosure of personal information.

Logs must **never include passwords, session tokens, message bodies, or other authentication secrets**.

#### RTCA038 — Built-in log rotation

Postela must manage its own log-file rotation by size.

The log directory, maximum size per file, and maximum number of retained files must be configurable. Old files must be deleted according to the configured retention limit.

Concurrent goroutine logging and rotation must be coordinated safely. Compression and an external rotation dependency are not required for the initial release.

### Service lifecycle and deployment

#### RTCA039 — Startup validation and initialization

The backend must validate its configuration and initialize required internal resources before accepting client connections.

A critical configuration or initialization failure must be logged and must prevent the service from starting.

Startup must **not depend on the availability of any configured IMAP, SMTP, or ManageSieve server**. Mail-server connections are established when needed for authentication, monitoring, or operations.

#### RTCA040 — Graceful shutdown

On an intentional shutdown, the backend must stop accepting new connections and requests, allow in-flight operations to finish within a configurable maximum shutdown period, and notify connected clients that the service is terminating.

It must then close WebSockets and persistent IMAP connections in an orderly manner, release resources, and exit.

Because sessions reside only in memory, clients must authenticate again after the service restarts. An intentional shutdown must not be presented as an ordinary transient WebSocket failure.

#### RTCA041 — Liveness and readiness endpoints

The backend must expose unauthenticated HTTP(S) endpoints for:

- **Liveness:** whether the process is running and able to respond.
- **Readiness:** whether initialization has completed and the service can accept connections.

Responses must disclose only minimal status information, never account data, session details, or configuration secrets.

Neither check may require external IMAP, SMTP, or ManageSieve servers to be available.

#### RTCA042 — Reverse-proxy-compatible deployment

Postela must support direct deployment and deployment behind a reverse proxy without depending on a specific proxy product.

A trusted proxy may terminate TLS for public HTTPS and WSS connections. Plain HTTP and WebSocket connections to the backend must be permitted **only on explicitly configured internal interfaces or trusted networks**, not exposed indiscriminately.

The backend must accept forwarded headers such as `X-Forwarded-For` and `X-Forwarded-Proto` only from configured trusted proxies.

Its public external URL must be explicitly configurable, **including any dedicated external port**, and the proxy must support the WebSocket upgrade handshake.

## Boundaries and deferred decisions

- **Sync and notifications (`RTSN`):** IMAP capability requirements, `NOTIFY PERSONAL` / `CONDSTORE`, parsing unsolicited events, three-attempt reconnection behavior, controlled close reasons, and state restoration are defined in the [sync and notifications document](./sync-and-notifications.md). RTCA defines shared-account session and resource ownership; it does not replace RTSN.
- **Data storage:** the storage model for Postela's own application data belongs to the Data storage requirements. Mail messages must not be persisted by Postela.
- **Mail operations:** message retrieval, modification, sending, and filtering belong to their respective requirements groups, and must follow the session and resource limits defined here.
- **Not yet fixed:** concrete configuration values and defaults, retry and timeout durations, the exact JSON protocol schema and error code catalog, specific WebSocket close codes, and the disconnected-session retention duration. These must be agreed before implementation when they materially affect behavior.
