# Realtime Chat Learning Project — Design Specification

Date: 2026-08-01  
Status: Approved design; implementation has not started

## 1. Purpose

Build a browser-based realtime communication application while teaching the learner to design and implement the system independently. The project uses React, strict TypeScript, Node.js, raw WebSockets, PostgreSQL, object storage, and WebRTC. The same protocol and domain boundaries should later be reusable from mobile clients and reimplementable with a Python backend.

The learner writes the application code. The teaching process supplies concepts, bounded exercises, review, debugging guidance, and verification. It does not silently implement features on the learner's behalf.

The initial schedule is eight days at approximately two focused hours per day. Every requested feature remains in scope, but understanding and correctness take priority over completing all features within sixteen hours.

## 2. Product Scope

### 2.1 Anonymous guests

- A guest may enter one shared public lobby without creating an account.
- The server assigns a distinguishable alias such as `Anonymous-4821`.
- The alias belongs to an opaque browser-session cookie and normally survives a WebSocket reconnect while the server process remains alive.
- Guests may send and receive text messages only in the public lobby.
- Guest messages and guest-session state remain in server memory. They disappear when the server restarts.
- Guests cannot create or join registered-user groups, open private conversations, upload documents, or place calls.

### 2.2 Registered users

- A user registers with a unique username and password.
- A registered user may use the public lobby.
- A registered user may discover, create, join, and leave public group rooms.
- Every registered user may join every group without an invitation or owner approval.
- The group creator is its owner. The owner's only special group capability is deleting that group.
- A registered user may start a private one-to-one conversation with another registered user.
- Private conversations contain exactly two registered users.
- Registered users may send PDF, DOCX, and TXT documents in private conversations and groups.
- Registered users may place audio or video calls only inside private one-to-one conversations.
- Group audio and video calls are not supported.

### 2.3 Explicitly excluded from the first version

- Email addresses, email verification, password reset, OAuth, and social login
- Profiles, avatars, friend requests, blocking, moderation, and administrative dashboards
- Invited or private groups
- Group audio/video calls, screen sharing, call recording, and call history
- Message editing, deletion, reactions, threads, read receipts, and typing indicators
- Image, archive, executable, audio, and video file attachments
- End-to-end encryption for stored text messages or documents
- Native mobile applications and the Python server rewrite; those are follow-up projects that reuse this design

## 3. Learning Contract

Each feature follows this loop:

1. Explain the relevant concept and its place in the system.
2. Ask the learner to describe the expected data flow and failure cases.
3. Give a small implementation exercise with acceptance criteria, not a completed solution.
4. Let the learner write the code.
5. Review the code for correctness, security, clarity, and TypeScript usage.
6. Run or have the learner run focused tests.
7. Ask the learner to explain the finished behavior without relying on notes.
8. Commit the verified slice before starting the next one.

Hints escalate from conceptual guidance, to pseudocode, to a minimal isolated example only when needed. A full feature solution is not supplied unless the learner explicitly changes this rule.

Each two-hour session targets approximately twenty minutes of instruction, fifteen minutes of learner-led design, sixty minutes of implementation, twenty minutes of testing/debugging, and five minutes of explanation from memory. These are guides rather than deadlines.

## 4. Architecture

The system is a TypeScript monorepo with three principal workspaces:

- `apps/web`: React and Vite browser client
- `apps/server`: Node.js HTTP and WebSocket application server
- `packages/protocol`: runtime schemas and inferred TypeScript types shared by client and server

Use Node.js 24 LTS, npm workspaces, strict TypeScript, and a committed lockfile. React state, context, and small custom hooks are sufficient; a global state-management framework is not required initially.

### 4.1 Transport boundaries

HTTPS handles request/response work:

- Registration, login, logout, and current-session lookup
- User lookup for starting a private conversation
- Group discovery, creation, joining, leaving, and deletion
- Paginated conversation history
- Document upload and authorized download

Raw WebSocket handles live state:

- Lobby and registered-conversation messages
- Connection readiness and presence needed for private calls
- Request acknowledgements and structured protocol errors
- WebRTC invitation, acceptance, rejection, signaling, and termination events

WebRTC handles audio/video media:

- Audio and video travel directly between two browsers when possible.
- STUN discovers usable network paths.
- TURN relays encrypted media when direct peer connectivity fails.
- The Node application carries signaling messages but never receives or processes media frames.

### 4.2 Server boundaries

The Node server is separated into small units:

- HTTP controllers translate requests and responses.
- The WebSocket gateway parses envelopes, authenticates connections, routes events, and sends acknowledgements.
- Runtime schemas validate all network input.
- Domain services implement authentication, conversations, permissions, messaging, attachments, and calling rules.
- Repositories contain parameterized PostgreSQL queries.
- Storage and TURN adapters isolate third-party providers behind application-owned interfaces.

WebSocket tells connected clients what changed. PostgreSQL remains the source of truth for durable application state.

## 5. Data Model

The PostgreSQL schema contains these core entities:

### `users`

- Immutable user ID
- Normalized unique username and display username
- Argon2id password hash
- Creation timestamp

### `sessions`

- Opaque session ID or a hash of the externally presented session token
- User ID
- Creation, last-used, and expiry timestamps
- Revocation timestamp when logged out

### `conversations`

- Conversation ID
- Type: `DIRECT` or `GROUP`
- Group name for groups; absent for direct conversations
- Owner user ID for groups; absent for direct conversations
- A canonical direct-pair key for direct conversations; absent for groups. It is formed from the two sorted user IDs and has a unique database constraint, preventing duplicate private conversations for the same pair.
- Creation timestamp

### `conversation_members`

- Conversation ID and user ID composite identity
- Join timestamp
- Direct conversations must contain exactly two members.
- Group membership is created explicitly when a registered user joins.

### `messages`

- Server-generated message ID
- Conversation ID
- Sender user ID
- Optional text body
- Server creation timestamp
- Unique client request ID scoped to the sender, used to prevent duplicate writes after reconnects

At least one of text body or attachment must be present. Durable messages exist only for direct and group conversations. The anonymous lobby uses an in-memory message structure instead.

### `attachments`

- Attachment ID
- Uploader user ID
- Message ID once attached
- Provider object identifier
- Original filename
- Validated media type
- Byte size
- Upload status and creation timestamp

An uploaded attachment begins unclaimed and may only be claimed once by its uploader. Failed or abandoned uploads are deleted best-effort and reported for cleanup if provider deletion fails.

Database constraints cover uniqueness and referential integrity. Application transactions enforce cross-row invariants such as exactly two direct-conversation members and one-time attachment claiming.

## 6. Authentication and Authorization

### 6.1 Credentials

- Usernames are 3–24 characters and contain letters, numbers, and underscores.
- Username uniqueness is case-insensitive.
- Passwords are 10–128 characters.
- Passwords are hashed with Argon2id using a maintained Node package. The initial minimum is 19 MiB memory, two iterations, and one degree of parallelism; deployment benchmarking may increase, but not reduce, these values.
- Passwords, raw session tokens, provider secrets, and TURN credentials are never logged.

### 6.2 Sessions

- Successful registration or login creates a cryptographically random server session.
- The browser receives the session identifier in a Secure, HttpOnly, SameSite=Lax cookie. `Secure` is enabled outside local HTTP development.
- Sessions expire after seven days and are revoked on logout.
- Login rotates the session identifier.
- State-changing HTTP requests require an allowed same-origin `Origin` header in addition to the SameSite cookie policy.
- The WebSocket upgrade authenticates from the same cookie. Identity is attached to server connection context and is not accepted from event payloads.
- Failed login attempts are limited to five per normalized-username-and-IP pair per fifteen minutes, with a broader limit of thirty attempts per IP per fifteen minutes. Registration is limited to ten attempts per IP per hour. Limits may become stricter after measurement, but not looser during the learning release.

### 6.3 Permission rules

- Guest connections may use only lobby join and lobby text-message events.
- Registered conversation history and live events require current membership.
- Any registered user may list groups and join an existing group.
- Only a group owner may delete that group.
- Only the two direct-conversation members may read or write its messages, download its documents, or exchange its call signaling.
- Every permission is checked on the server for every relevant HTTP request and WebSocket event. Hiding controls in React is a usability measure, not a security boundary.

## 7. WebSocket Protocol

All client requests use a versioned JSON envelope containing:

- Protocol version
- Event type
- Client-generated request ID
- Event-specific payload

Server responses are one of:

- Acknowledgement correlated to the request ID
- Canonical domain event containing server-selected IDs, identity, and timestamp
- Structured error containing a stable error code, a safe message, and a retryable flag

Initial event families are:

- Connection: ready, ping, pong, protocol error, and authentication expiry
- Lobby: join, leave, and text message
- Conversations: subscribe, unsubscribe, and message send/created
- Calls: invite, accept, reject, offer, answer, ICE candidate, end, and timeout

Event payloads are validated at runtime using schemas from `packages/protocol`. TypeScript's compile-time types are inferred from those schemas but are never treated as runtime validation.

Text is trimmed, must contain 1–2,000 Unicode characters, and is rendered as text rather than injected HTML. WebSocket frames are limited to 16 KiB because document bytes use HTTP. A per-session token bucket allows a short burst of five messages and replenishes at one message per second.

The server commits a durable message before broadcasting it. A successful acknowledgement therefore means the message is stored. Repeating a message request with the same user-scoped request ID returns the original result rather than inserting a duplicate.

After reconnecting, the client uses HTTP history pagination to recover durable messages that it may have missed, then resumes live WebSocket subscription. Guest messages cannot be recovered after server loss.

## 8. Feature Data Flows

### 8.1 Guest connection

1. The browser requests an anonymous session over HTTP.
2. The server creates an in-memory session, a random alias, and an opaque session cookie.
3. The WebSocket upgrade presents that cookie.
4. The server sends connection-ready state containing the alias and guest capabilities.
5. Lobby messages are validated and broadcast in memory without database persistence.

### 8.2 Registered message

1. The client sends a message event with conversation ID, request ID, text, and optional attachment ID.
2. The gateway validates the envelope and resolves the authenticated user from connection context.
3. The service verifies membership and attachment ownership.
4. A transaction inserts the message and claims the attachment.
5. The server acknowledges the sender and broadcasts one canonical message to currently subscribed members.

### 8.3 Document

1. An authenticated member uploads one document over multipart HTTPS.
2. The server enforces a 5 MB limit and accepts only PDF, DOCX, and plain text.
3. Validation considers filename extension, declared MIME type, and content. PDF data must begin with a valid PDF signature. DOCX data must be a valid ZIP package containing the required Office document entries. TXT data must decode as UTF-8 and contain no NUL bytes. Filename and MIME type alone are insufficient.
4. The server streams the file to private provider storage rather than the Render filesystem.
5. The resulting unclaimed attachment ID may be referenced by one subsequent message from the uploader.
6. Download requests recheck conversation membership and return a short-lived signed provider URL.

### 8.4 Private call

1. A registered caller sends an invitation for a direct conversation.
2. The server verifies membership, confirms the other member is online, and creates ephemeral call state.
3. The recipient accepts or rejects. Unanswered invitations expire after thirty seconds.
4. Accepted peers exchange SDP offers/answers and ICE candidates only through authorized WebSocket events.
5. Audio/video travels through WebRTC, using TURN only when necessary.
6. Either peer may end the call. Disconnects and signaling timeouts clean up ephemeral call state.

Only one active call is allowed per direct conversation in the first version. Calls are not recorded or stored in history.

## 9. User Interface

The application uses one responsive chat shell:

- A left navigation area contains the anonymous lobby, discoverable groups, joined groups, and direct conversations.
- The central area contains the selected conversation, message history, connection status, and composer.
- A details area contains conversation-specific information and permitted actions; it may collapse on narrow screens.
- Guests see only the lobby, their generated alias, text composer, and registration/login entry points.
- Registered users see group and direct-message navigation plus document controls.
- Private-conversation headers show audio and video actions.
- Group headers never show call actions.
- Group owners see a delete action with an explicit destructive confirmation.

The React client renders controls from server-provided capabilities or locally derived authenticated state, while still expecting the server to reject unauthorized requests. Loading, empty, offline, reconnecting, upload-progress, and error states are first-class UI states.

## 10. Error Handling and Resilience

- Malformed JSON and invalid payloads produce structured protocol errors and do not crash the process.
- Unsupported protocol versions are rejected explicitly.
- Expected authentication, authorization, validation, conflict, offline-recipient, and rate-limit errors use stable codes.
- Internal error details are logged with request IDs but never sent to clients.
- HTTP and WebSocket requests have correlation IDs for diagnosis.
- Ping/pong heartbeats detect dead connections.
- Clients reconnect with exponential backoff, jitter, and an upper delay bound.
- Reconnect restoration is idempotent and re-fetches missed durable history.
- Database writes complete before successful acknowledgement or broadcast.
- Upload streams have byte limits and cleanup behavior for partial failure.
- Call invitations and signaling phases have timeouts and deterministic cleanup.
- Process-level handlers log unexpected failures and allow the platform to restart the service; they do not pretend the corrupted operation succeeded.

## 11. Testing Strategy

Testing proceeds from inexpensive isolated checks to full workflows:

1. Vitest unit tests for runtime schemas, normalization, protocol routing, and pure permission rules.
2. PostgreSQL integration tests for registration, sessions, membership, direct-conversation uniqueness, message idempotency, ownership, and attachment claiming.
3. HTTP integration tests for authentication, group lifecycle, history pagination, upload rejection, and download authorization.
4. WebSocket integration tests using multiple real clients for guest restrictions, room isolation, acknowledgements, broadcasts, reconnect behavior, and call-signaling authorization.
5. React Testing Library tests for capability-driven controls, connection states, errors, and accessible interaction.
6. Playwright browser tests for registration, private chat, groups, and document sharing.
7. Manual two-browser and two-device WebRTC checks for permissions, acceptance/rejection, audio/video toggles, disconnect cleanup, and TURN fallback.

Security tests intentionally modify client payloads and IDs to prove that UI restrictions cannot bypass server authorization.

## 12. Free Deployment Design

### Application server: Render

One Render Web Service serves the compiled React application, HTTPS API, and WebSocket endpoint from the same origin. This avoids cross-origin cookie and WebSocket-authentication complexity. Render supports WebSockets and provides a free service suitable for a learning project.

Known limitation: a free service spins down after fifteen minutes without inbound HTTP or WebSocket traffic and can take about one minute to restart. Its filesystem is ephemeral, so it never stores durable database or uploaded-file state.

References: [Render free services](https://render.com/docs/free), [Render WebSockets](https://render.com/docs/websocket)

### Database: Neon PostgreSQL

Neon's free PostgreSQL plan currently includes 0.5 GB storage and 100 compute-unit hours per month per project with scale-to-zero behavior. The server uses the pooled connection endpoint and explicit parameterized SQL through the `pg` driver.

Reference: [Neon pricing](https://neon.com/pricing)

### Documents: Cloudinary

Cloudinary's no-credit-card free plan currently provides 25 monthly credits shared across storage and bandwidth. DOCX and TXT files are stored as raw private assets. PDFs are also stored as raw private assets without transformations. During deployment, enable PDF delivery in Cloudinary's Security settings, keep every document private, and verify that an unsigned public URL fails. Downloads use time-limited signed URLs created only after application authorization.

The application depends on its own storage interface rather than Cloudinary-specific types outside the adapter, making a later move to S3-compatible storage a focused change.

References: [Cloudinary free plan](https://cloudinary.com/documentation/billing_and_plans), [raw and private assets](https://cloudinary.com/documentation/upload_parameters), [private download URLs](https://cloudinary.com/documentation/image_upload_api_reference)

### WebRTC connectivity: Metered Open Relay

Metered Open Relay currently supplies free STUN/TURN service with 20 GB of relay traffic per month. Credentials are retrieved by the server and are not committed to the repository.

Reference: [Metered Open Relay](https://www.metered.ca/tools/openrelay/)

The deployment is expected to cost zero while usage remains inside current provider limits. Free tiers are intended for learning and demonstrations, may impose sleep or quota behavior, and may change in the future.

### Why not Vercel initially

Vercel's June 2026 WebSocket support pins each connection to a temporary Function instance. Connections end at the Function's maximum duration, and different users may land on different instances, requiring reconnect logic and shared Redis pub/sub for consistent rooms and presence. That distributed serverless model is a valuable later exercise but obscures the persistent-server fundamentals targeted here. Vercel can later host the React frontend or a deliberately redesigned serverless version.

References: [Vercel WebSocket chat architecture](https://vercel.com/kb/guide/real-time-chat-websockets), [Vercel Function limits](https://vercel.com/docs/functions/limitations)

## 13. Eight-Day Learning Sequence

### Day 1 — TypeScript and protocol foundations

- Strict TypeScript, module boundaries, Node's event loop, HTTP versus WebSocket
- Shared runtime schemas and typed protocol envelopes
- Raw multi-client WebSocket connection and focused protocol tests

### Day 2 — Anonymous lobby

- React state and effects, connection lifecycle, cleanup, and immutable updates
- Guest session, distinguishable alias, lobby subscription, live text, and reconnection

### Day 3 — PostgreSQL and authentication

- Relational modeling, migrations, parameterized queries, transactions
- Password hashing, secure sessions, cookies, HTTP authorization, and WebSocket upgrade authentication

### Day 4 — Private and group conversations

- Private-conversation uniqueness and membership
- Group discovery, creation, open joining, leaving, owner deletion
- Persistent messages, pagination, authorization, and reconnect recovery

### Day 5 — Document sharing

- Multipart HTTP, streaming, size/type/signature validation
- Private provider storage, attachment lifecycle, authorized signed downloads

### Day 6 — Private WebRTC calls

- Media permissions, `RTCPeerConnection`, SDP, ICE, STUN/TURN
- Authenticated signaling, invitation state, audio/video controls, and cleanup

### Day 7 — Reliability and security

- Heartbeats, backoff and jitter, idempotency, rate limiting, structured errors
- Permission attacks, failure injection, integration tests, and focused refactoring

### Day 8 — Deployment and verification

- Production builds, environment variables, migrations, HTTPS/WSS, provider configuration
- Two-browser/two-device workflow tests, logs, cold-start behavior, and failure drills

Unfinished work carries forward. No feature is completed by pasting a hidden solution merely to preserve this schedule.

## 14. Acceptance Criteria

The complete project is successful when:

- Two guest browsers receive distinguishable aliases and exchange lobby text in realtime.
- A guest is rejected from every registered-only HTTP route and WebSocket event.
- A user can register, log in, log out, and restore a valid session without exposing credentials to browser JavaScript.
- Registered users can start one private conversation, exchange durable messages, reconnect, and recover missed history.
- Registered users can create, discover, join, leave, and chat in public groups.
- Only the group owner can delete a group.
- Members can upload and download authorized PDF, DOCX, and TXT attachments up to 5 MB in private and group conversations.
- Nonmembers cannot read messages, download documents, or exchange signaling for a conversation.
- Two private-conversation members can complete audio and video calls; group call attempts are rejected.
- Duplicate message requests do not create duplicate durable messages.
- Automated tests cover core validation, permissions, persistence, HTTP, and WebSocket behavior.
- The application deploys through GitHub to the documented free services and passes the critical two-browser workflows.
- The learner can explain and recreate the protocol, authentication boundary, persistence flow, and WebRTC signaling design without copying the original implementation.

## 15. Follow-Up Learning

After the TypeScript project, preserve the protocol and acceptance tests while replacing the Node server with Python. A suitable rewrite uses an ASGI framework, Python runtime validation, PostgreSQL, and the same HTTP/WebSocket event semantics. Mobile clients can then be added against either backend without changing the domain protocol.

An advanced deployment exercise can move the frontend to Vercel and redesign realtime coordination for serverless instances with shared pub/sub. That exercise is intentionally separate from the initial persistent-server implementation.
