# Web Coding Agent

## Technical design

| Field | Value |
| --- | --- |
| Status | Proposed |
| Date | 2026-10-20 |
| Runtime | Browser, Web Worker, QuickJS-WebAssembly |
| Agent harness | `@earendil-works/pi-durable` |
| Persistence | SQLite-WebAssembly on OPFS |

## Summary

Web Coding Agent is a local-first coding environment whose agent process runs
inside the browser. The Pi durable harness and model adapters execute inside an
asyncified QuickJS WebAssembly VM hosted by a dedicated Web Worker. A small
JavaScript host supplies explicit capabilities for model transport, durable
SQLite storage, project files, clocks, cryptography, and sandboxed command
execution.

The browser owns the agent, conversation state, task scheduler, and project.
The only required remote component is the chosen model API. A production
deployment may place a stateless credential relay in front of that API, but the
relay does not run the agent or store its workspace.

The design deliberately does not emulate a general Linux machine. Coding tools
operate on a virtual project filesystem and commands run as allowlisted Wasm or
WASI programs with no ambient filesystem, network, DOM, cookie, or credential
access.

## Context

`pi-durable` provides a durable agent harness rather than a complete browser
application. Its conversations are immutable transcripts, and its model and
tool work is represented by checkpointed tasks. It supports host-provided
storage and execution environments, but its standard persistent storage and
coding environment adapters use Node.js.

The browser supplies the primitives needed to replace those adapters:

- Web Workers isolate the runtime from the UI and keep the main thread
  responsive.
- QuickJS compiled to WebAssembly can execute the bundled TypeScript/JavaScript
  runtime without giving it ambient browser authority.
- SQLite compiled to WebAssembly supplies transactions.
- OPFS supplies origin-private persistent bytes to SQLite.
- Web Locks and `BroadcastChannel` coordinate multiple tabs.
- WASI runners provide constrained command execution for precompiled tools.

The principal complication is that Pi, SQLite, project files, model streaming,
and the UI cross different runtime boundaries. This document defines those
boundaries and makes crash behavior explicit.

## Goals

- Run `pi-durable` inside a WebAssembly JavaScript VM in the browser.
- Preserve committed conversations, task checkpoints, tool effects, and project
  files across refreshes, worker crashes, and browser restarts.
- Support useful coding operations: inspect files, search, edit, review diffs,
  and run a constrained set of checks.
- Keep the UI responsive while the model, agent, database, and commands run.
- Give users a clear approval boundary before destructive or externally visible
  effects.
- Keep model credentials and application credentials outside the agent VM.
- Make the complete project and session exportable and restorable.

## Non-goals

- General POSIX or Linux compatibility.
- Running arbitrary native binaries or package-manager install scripts.
- Unattended execution after the browser has suspended or closed the page.
- Multi-user real-time collaboration.
- Transparent access to an unrestricted local checkout.
- Compatibility with VS Code extensions, containers, or arbitrary language
  servers.
- Hiding browser storage quotas or lifecycle limitations from the user.

## Requirements

### Functional

- A user can create, open, stop, resume, export, import, and reset a project.
- A user can submit a prompt and observe committed agent and tool activity.
- The agent can call `read`, `list`, `search`, `write`, `edit`, and `run`.
- Every file mutation has a stable operation identifier and a reviewable diff.
- A restarted runtime resumes Pi's pending work from its last checkpoint.
- A second tab cannot become a concurrent writer to the same project.
- The runtime supports an OpenAI-compatible streaming model endpoint.

### Safety and reliability

- The agent cannot access the DOM, browser cookies, origin storage outside its
  host API, or paths outside the active project.
- A repeated tool request with the same operation identifier has at most one
  committed project effect.
- An interrupted external effect is reported as uncertain rather than guessed
  successful.
- Secrets are absent from durable storage, exports, logs, and agent-visible
  configuration.
- Clearing site data is the only browser action allowed to silently remove the
  underlying OPFS; the application warns about this limitation and offers
  export.

## Architecture decisions

### AD-1: Pi executes in asyncified QuickJS-Wasm

The Pi bundle runs inside `quickjs-emscripten` in a dedicated module worker.
The production build uses a release, asyncified QuickJS Wasm variant because
host operations such as model streaming and SQLite are asynchronous.

The Pi bundle is compiled into one self-contained script before it is loaded
into QuickJS. It must not use dynamic imports or Node built-ins. Node-specific
Pi adapters are excluded and replaced with browser adapters in the bundle.

QuickJS receives exactly one host function:

```ts
declare function hostCall(requestJson: string): Promise<string>;
```

All authority is expressed as versioned JSON requests through this function.
The VM has explicit memory and execution limits and is disposed when the worker
loses project ownership.

This choice satisfies the requirement that Pi itself run under WebAssembly. It
costs more bundle size and bridge complexity than running Pi directly in the
worker. Native worker JavaScript remains a diagnostic build target, not an
automatic production fallback.

### AD-2: one worker and one writer own an open project

One agent worker owns one open project. Before opening SQLite, it acquires the
exclusive Web Lock `wca:project:<project-id>`. The lock remains held for the
worker lifetime.

Other tabs are observers. The owner broadcasts sanitized snapshots and status
over `BroadcastChannel`. An observer may request takeover, but it cannot open a
writable connection until the current owner closes and releases the lock.

This matches Pi's single-owner storage model and avoids cross-tab database
coordination inside the QuickJS VM.

### AD-3: SQLite is the canonical store for both Pi and project files

One SQLite-Wasm database per project is persisted in OPFS. Pi uses namespaced
tables with the prefix `pi_`; application tables use `app_`.

Project files are rows in SQLite rather than independent OPFS files. This gives
file mutations, revisions, operation deduplication, and audit records a common
transaction boundary. Command workers receive a materialized snapshot and
return a candidate delta; they do not directly edit the canonical project.

OPFS is an implementation detail of the database. Users interact with project
imports and exports, not with the opaque OPFS layout.

### AD-4: tool effects are idempotent and separately journaled

Pi commits tool intent before execution, but Pi's result commit occurs after
the external tool effect. The host therefore maintains an application effect
ledger keyed by Pi's stable tool-call identifier.

For a project mutation, one SQLite transaction:

1. checks whether the operation identifier already completed;
2. validates the hash of the arguments;
3. applies file mutations against expected file revisions;
4. records the result and resulting revisions; and
5. marks the operation complete.

A replay with the same identifier and argument hash returns the stored result.
A replay with different arguments is a protocol error. This makes local file
tools safe to replay even if the worker dies before Pi commits the tool result.

Networked and other non-transactional effects are not replay-safe. The MVP does
not expose them to the agent.

### AD-5: commands are capabilities, not a shell

The `run` tool accepts a structured command selected from a host registry:

```ts
interface RunRequest {
  command: string;
  args: string[];
  cwd: string;
  timeoutMs: number;
}
```

There is no shell parsing, command substitution, redirection, inherited
environment, or ambient network. Each registered command defines:

- its exact Wasm/WASI module and integrity hash;
- argument validation;
- preopened directories;
- whether project input is read-only;
- CPU/fuel, wall-clock, memory, and output limits; and
- whether any resulting file delta may be offered for approval.

The initial registry contains search, formatting, and a JavaScript checker.
Language runtimes are added independently after their sandbox and persistence
behavior is specified.

### AD-6: credentials remain in the JavaScript host

The QuickJS VM receives an opaque model configuration identifier, never an API
key. `hostCall` starts and advances model requests through the host transport.

Production uses a same-origin credential relay. The relay authenticates the
browser, injects the provider key, preserves streaming and cancellation, and
applies quotas. It stores no conversation or project state.

A development BYOK mode may hold a user key in host memory for the tab lifetime.
It is never written to browser storage or forwarded into the VM.

## System structure

```text
Browser tab
|
|  commands / committed snapshots
v
UI main thread
|
|  typed postMessage protocol
v
Agent module worker  <---- Web Lock / BroadcastChannel ----> observer tabs
|
|-- Runtime controller
|     |-- QuickJS-Wasm VM
|     |     `-- bundled pi-durable + browser adapters
|     |-- hostCall capability router
|     |-- model stream manager
|     `-- SQLite-Wasm connection
|
|-- OPFS
|     `-- one SQLite database per project
|
`-- execution worker pool
      `-- allowlisted Wasm/WASI command + ephemeral workspace snapshot

Optional same-origin relay ----> model provider
```

### UI main thread

The UI renders a projection of durable runtime state. It never mutates the
database and never calls a model provider directly.

Its responsibilities are:

- session and project selection;
- transcript, editor, diff, approval, and run-output presentation;
- forwarding user commands with unique request identifiers;
- retaining only replaceable view preferences in normal browser storage; and
- reconnecting by requesting a fresh snapshot rather than replaying missed UI
  events.

### Runtime controller

The controller owns worker initialization and all host authority. It:

- acquires and holds the project lock;
- opens SQLite-Wasm and performs application schema migration;
- creates the QuickJS VM and installs `hostCall`;
- evaluates the pinned Pi bundle;
- serializes entry into the asyncified VM;
- validates every VM request and UI command;
- manages model and command abort controllers; and
- tears down resources in reverse ownership order.

QuickJS must never be entered concurrently. Model chunks, tool completions, and
UI commands enter a FIFO mailbox. Long-running host work may happen in parallel,
but resolution back into the VM is serialized.

### Pi browser bundle

The bundle contains:

- `pi-durable` and its required Pi AI/Chord dependencies;
- a host-backed `SqliteDatabase` adapter;
- an OpenAI-compatible provider implemented over the model bridge;
- a browser coding extension defining the supported tools;
- the project-specific system prompt and safety policy; and
- a watcher that converts durable Pi state into UI-safe snapshots.

It excludes Node filesystem, process, SQLite, JSONL, and execution adapters.
Every upstream package is pinned to an exact version and lockfile integrity.

### SQLite host

SQLite-Wasm runs in the agent worker and persists through OPFS. The bridge
exposes prepared statement operations and explicit transaction handles:

```ts
type SqlRequest =
  | { op: "begin" }
  | { op: "exec"; tx?: string; sql: string }
  | { op: "run"; tx?: string; sql: string; bindings: JsonValue[] }
  | { op: "get"; tx?: string; sql: string; bindings: JsonValue[] }
  | { op: "all"; tx?: string; sql: string; bindings: JsonValue[] }
  | { op: "commit"; tx: string }
  | { op: "rollback"; tx: string };
```

Only the Pi storage adapter may issue arbitrary SQL, and it is confined to the
`pi_` namespace by construction. Application operations call named host methods
rather than submitting SQL from the VM. A transaction handle owns the database
queue until commit or rollback. Worker termination closes the connection and
rolls back an open transaction.

### Project service

The project service is implemented in the host and exposes named operations:

- `project.list`
- `project.read`
- `project.search`
- `project.applyEdit`
- `project.applyBatch`
- `project.snapshot`
- `project.export`

Paths are UTF-8, slash-separated, relative paths in normalized NFC form. Empty
segments, `.`, `..`, NUL, absolute roots, backslashes, and platform drive
prefixes are rejected. Case sensitivity is consistent across browsers because
paths are database keys rather than host filesystem paths.

### Execution workers

Commands run outside the agent worker so a slow or trapped Wasm program cannot
block Pi's scheduler. The runtime controller creates a fresh execution worker
for each command or reuses a worker only after the runtime has proven it can be
fully reset.

The controller materializes the requested project revision into the command's
virtual filesystem. The default project mount is read-only and `/tmp` is
writable. Standard output and error are streamed with fixed byte limits.
Timeout or cancellation terminates the worker.

If a command is permitted to propose changes, the worker returns a file delta.
The host validates and displays the delta, and only a separate approved
`project.applyBatch` operation commits it.

## External protocols

### UI-to-worker commands

Every command has a unique `requestId`. Duplicate identifiers return the
original response while it remains in the runtime request cache.

```ts
type UiCommand =
  | { type: "open"; requestId: string; projectId: string }
  | { type: "submit"; requestId: string; text: string }
  | { type: "stop"; requestId: string }
  | { type: "approve"; requestId: string; approvalId: string }
  | { type: "reject"; requestId: string; approvalId: string; reason?: string }
  | { type: "readFile"; requestId: string; path: string }
  | { type: "export"; requestId: string };
```

Messages are validated against a runtime schema before dispatch. Unknown
fields are rejected on authority-bearing commands.

### Worker-to-UI events

```ts
type RuntimeEvent =
  | { type: "snapshot"; revision: number; state: PublicRuntimeState }
  | { type: "status"; activity: Activity }
  | { type: "approvalRequired"; approval: PublicApproval }
  | { type: "commandOutput"; commandId: string; stream: "stdout" | "stderr"; chunk: string }
  | { type: "response"; requestId: string; result: JsonValue }
  | { type: "error"; requestId?: string; error: PublicError };
```

Snapshots are authoritative and monotonically revisioned. Status and command
output are ephemeral. Reconnection always begins with a snapshot, so correctness
does not depend on replaying UI events.

### QuickJS host capabilities

The JSON request envelope contains protocol version, method, call identifier,
and parameters. The router uses a static method table; names cannot select
arbitrary JavaScript properties.

```ts
interface HostRequest {
  version: 1;
  callId: string;
  method:
    | "clock.now"
    | "crypto.randomUUID"
    | "model.open"
    | "model.next"
    | "model.abort"
    | "sql.begin"
    | "sql.exec"
    | "sql.run"
    | "sql.get"
    | "sql.all"
    | "sql.commit"
    | "sql.rollback"
    | "project.list"
    | "project.read"
    | "project.search"
    | "project.applyEdit"
    | "command.run"
    | "command.abort"
    | "ui.publish";
  params: JsonValue;
}
```

The host response is either `{ ok: true, value }` or a stable, sanitized error
object. Native exceptions and stack traces do not cross the boundary.

### Model streaming

The Pi provider adapter calls `model.open`, which returns an opaque stream
identifier. It repeatedly awaits `model.next`; each result is one normalized
provider event or an end marker. `model.abort` is idempotent.

The host validates the target model against user settings, attaches credentials,
performs the fetch, parses the provider stream, and maps provider errors into a
stable taxonomy. Raw response headers and credential-bearing request data never
enter QuickJS.

## Data model

Pi owns its namespaced tables. The application owns the following logical
schema; exact column types may change without changing the boundary.

```sql
CREATE TABLE app_meta (
  key TEXT PRIMARY KEY,
  value_json TEXT NOT NULL
);

CREATE TABLE app_project (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  revision INTEGER NOT NULL
);

CREATE TABLE app_file (
  project_id TEXT NOT NULL,
  path TEXT NOT NULL,
  content BLOB NOT NULL,
  content_hash TEXT NOT NULL,
  revision INTEGER NOT NULL,
  mode INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (project_id, path)
);

CREATE TABLE app_effect (
  operation_id TEXT PRIMARY KEY,
  kind TEXT NOT NULL,
  arguments_hash TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('pending', 'done', 'uncertain')),
  result_json TEXT,
  created_at INTEGER NOT NULL,
  finished_at INTEGER
);

CREATE TABLE app_approval (
  id TEXT PRIMARY KEY,
  operation_id TEXT NOT NULL,
  summary_json TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('pending', 'approved', 'rejected')),
  decided_at INTEGER
);
```

`app_project.revision` increments once for each committed file transaction.
Every edit includes expected file revisions. A stale edit returns a conflict and
does not partially apply.

File contents are limited per file and per project. Binary files may be stored
and exported, but agent text tools reject unsupported encodings instead of
silently transforming them.

## Core flows

### Open and resume

1. The UI starts an agent worker and sends `open`.
2. The controller acquires the project's Web Lock.
3. It opens the SQLite-Wasm database from OPFS and migrates `app_` tables.
4. It instantiates QuickJS-Wasm, installs `hostCall`, and evaluates the Pi
   browser bundle.
5. The Pi storage adapter opens the `pi_` namespace.
6. Pi reconstructs the conversation and resumes runnable durable tasks.
7. The watcher publishes a complete committed snapshot to the UI.

If any step fails, later resources are not opened. Teardown closes Pi, disposes
QuickJS, closes SQLite, and releases the lock.

### User submission and model turn

1. The UI submits text with a request identifier.
2. Pi admits the input using that identifier for deduplication.
3. Pi's generation task asks the browser provider to open a model stream.
4. The host fetches the provider and yields normalized events through
   `model.next`.
5. Pi commits its durable entries/checkpoints.
6. The watcher publishes the resulting committed snapshot.

The UI may show an ephemeral “model running” status, but transcript content is
rendered only from Pi's committed state.

### File edit tool

1. Pi commits the tool task and invokes `project.applyEdit` with its stable call
   identifier.
2. The host normalizes the path and hashes the canonical arguments.
3. It checks `app_effect` for the identifier.
4. If already done with the same hash, it returns the saved result.
5. Otherwise it validates expected revisions, computes the diff, and applies
   the edit plus effect result in one transaction.
6. Pi receives the result and commits its tool result.

If the worker dies between steps 5 and 6, Pi may replay the tool. Step 4 returns
the already committed result without editing the file again.

### Command execution

1. Pi requests a registered command.
2. The host validates command, arguments, working directory, and policy.
3. If approval is required, the durable agent task waits while the UI displays
   an approval record.
4. The project service creates an immutable snapshot at a known revision.
5. A fresh execution worker starts the integrity-pinned Wasm module with limits.
6. Output is streamed ephemerally and the final bounded result returns to Pi.
7. Any proposed file delta requires a separate approved project transaction.

Read-only commands are replay-safe. Commands with external effects are absent
from the MVP registry.

### Crash recovery

- If the UI dies, the worker may continue until the browser terminates it; a
  reconnected owner requests a fresh snapshot.
- If the agent worker dies, SQLite rolls back open transactions and the Web Lock
  is released.
- A new worker reopens Pi and calls resume. Pi continues checkpointed tasks.
- A replayed local effect is resolved from `app_effect`.
- A terminated command has no canonical file authority and is safe to rerun.
- A model request interrupted before its response is committed may be issued
  again; model requests therefore use provider idempotency when available and
  are treated as cost-at-least-once.

## Approval model

The host, not the prompt or agent, decides whether approval is required.
Policies match normalized capability requests and can only narrow authority.

Always require approval for:

- deleting or renaming files;
- a batch exceeding configured file or byte thresholds;
- replacing a file after a revision conflict;
- accepting command-produced file changes; and
- any future network or external-account capability.

Read-only operations need no approval. Simple writes may be configured for
automatic approval within the mounted project, but the UI still exposes the
resulting diff and audit record.

An approval is bound to the operation identifier, argument hash, project
revision, and concise effect summary. Changing any of them invalidates it.

## Security model

### Trust boundaries

- **Untrusted:** model output, project contents, imported archives, tool
  arguments, command output, and Wasm command modules not shipped by the app.
- **Constrained:** Pi bundle in QuickJS-Wasm and allowlisted commands in
  execution workers.
- **Trusted:** runtime controller, capability router, schema validators,
  project service, SQLite host, and deployment configuration.
- **Secret-bearing:** model transport and credential relay only.

### Controls

- QuickJS has no direct `window`, DOM, `fetch`, WebSocket, cookies, storage,
  clipboard, or service-worker APIs.
- The capability router validates method-specific schemas, sizes, and ownership.
- Commands get no network imports and only explicit virtual filesystem mounts.
- Wasm modules are self-hosted and checked against build-time integrity hashes.
- CSP denies inline scripts, unlisted connections, plugins, and framing.
- COOP/COEP headers provide cross-origin isolation where required by the chosen
  SQLite/Wasm configuration.
- Imported archives enforce entry-count, uncompressed-size, compression-ratio,
  path, and filename limits before committing files.
- Tool and model output is length-limited and treated as text, never HTML.
- Telemetry excludes prompts, model output, file contents, tool arguments,
  credentials, and raw provider errors.

Wasm is one containment layer, not the sole security boundary. All host calls
remain hostile inputs even when they originate from the shipped Pi bundle.

## Storage, backup, and migration

The application requests persistent browser storage and displays the result of
`navigator.storage.persisted()` plus current quota estimates. It cannot promise
survival after the user clears site data.

An export contains:

- a consistent SQLite database snapshot;
- a JSON manifest with format, application, Pi package, and schema versions;
- integrity hashes; and
- no secrets.

Export briefly pauses new agent submissions and uses SQLite's backup/snapshot
mechanism rather than copying a live OPFS file. Import validates the entire
archive into a new project identifier before making it visible. It never
overwrites an existing project in place.

Application migrations are forward-only, transactional, and recorded in
`app_meta`. Pi schema migrations remain owned by the pinned Pi storage package.
An application upgrade takes a backup snapshot before the first irreversible
migration.

## Failure handling

| Failure | Behavior |
| --- | --- |
| Web Lock unavailable | Open as observer and offer explicit takeover request. |
| OPFS unavailable | Refuse durable mode; optionally offer a clearly labeled ephemeral demo. |
| Storage quota exhausted | Stop new writes, keep the database open for export, and report required action. |
| SQLite transaction interrupted | Roll back; resume from the last committed Pi/app state. |
| QuickJS memory/CPU limit | Terminate the VM, preserve SQLite, and offer restart/resume. |
| Command timeout | Terminate its execution worker and return a bounded timeout result. |
| Model stream disconnect | Let Pi/provider retry policy decide; retain committed state and expose cost uncertainty. |
| Credential relay unavailable | Keep the project usable and queued work visible; do not lose or fabricate an answer. |
| File revision conflict | Apply nothing and return current revision plus a conflict summary. |
| Database migration failure | Roll back and leave the previous version exportable. |

Errors exposed to the UI use stable codes, safe messages, retryability, and a
correlation identifier. Raw exceptions remain in local development diagnostics.

## Performance constraints

- UI handlers must not perform database, model, archive, or Wasm execution work.
- QuickJS heap, command memory, output, file, project, and transcript limits are
  configured independently.
- Runtime snapshots are incremental during normal operation, with periodic full
  snapshots for reconnection.
- Large file contents are fetched on demand and are not embedded in session
  snapshots.
- Search and list operations execute against indexed application data and return
  bounded, paginated results.
- Only one call may re-enter QuickJS at a time; the mailbox exposes queue depth
  so overload is visible rather than expressed as UI hangs.

Initial release budgets are a warm resume under two seconds for a typical
project, main-thread tasks under 50 ms, and a default combined runtime memory
ceiling of 512 MiB. These are product budgets, not claims about every device.

## Deployment

The application is served over HTTPS with immutable hashed assets. QuickJS,
SQLite, command modules, and worker scripts are served from the same origin.

Required response policy includes:

```text
Content-Security-Policy: default-src 'self'; script-src 'self'; worker-src 'self'; connect-src 'self' <configured-model-or-relay>; object-src 'none'; base-uri 'none'; frame-ancestors 'none'
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

The service worker caches only versioned application assets. It does not cache
model traffic, credentials, project exports, or runtime API responses. An
upgrade activates only after active workers have reached a safe shutdown point
or the user explicitly reloads.

## Observability

Local diagnostics record durations, counters, state transitions, package
versions, Wasm memory high-water marks, and stable error codes. Content fields
are excluded.

Optional remote telemetry is opt-in and contains only aggregate operational
metrics. Correlation identifiers are newly generated and are not project,
conversation, provider, or user identifiers.

The diagnostics export is separate from the project export and presents its
exact contents for review before download.

## Verification

The design is considered valid when the implementation demonstrates:

- Pi executing inside QuickJS-Wasm, including a streaming model call and tool
  round trip;
- the Pi storage conformance suite passing through the host SQLite adapter;
- a worker kill after each durable boundary without lost committed state;
- repeated file-tool operation identifiers producing one project mutation;
- cross-tab writer exclusion and clean ownership takeover;
- termination of infinite-loop and over-memory command modules;
- path traversal, archive bomb, malformed protocol, and capability escape tests;
- restore of an export in a fresh browser profile; and
- browser tests in current Chromium, Firefox, and WebKit, with unsupported
  capabilities reported explicitly.

A deterministic fake model drives reliability tests. Live providers are used
only for a small opt-in compatibility suite.

## Alternatives considered

### Run Pi as native JavaScript in the worker

This is simpler, faster, and likely more compatible with upstream Pi. It is not
the selected design because it does not meet the requirement that Wasm run the
Pi agent. It remains useful as a diagnostic baseline: a failure reproduced in
both builds is unlikely to be caused by QuickJS.

### Run Pi on a backend

This makes Node tools and background operation straightforward, but the browser
would become a remote UI rather than the agent host. It conflicts with the
local-first objective and moves project trust to the server.

### Use WebContainers

WebContainers offer broader Node and shell compatibility but create a larger,
vendor-specific runtime and deployment constraint. They also weaken the design's
small explicit capability surface. They may become an optional execution
backend, not the durable core.

### Store project files as loose OPFS entries

This maps more naturally to a filesystem but cannot atomically commit the file
effect ledger with content changes. SQLite rows provide clearer crash and replay
semantics. Export can reconstruct a normal directory tree.

### Use IndexedDB or JSONL for Pi state

Both can persist browser data, but SQLite already matches Pi's transactional
storage abstraction and can atomically own application metadata and files.
JSONL also grows with history and complicates indexed project operations.

### Expose a general WASI shell

This appears familiar but overstates browser compatibility and creates a broad
authority surface. Structured, registered commands are easier to validate,
limit, explain, and reproduce.

## Open questions

- Does the pinned Pi dependency graph execute correctly under QuickJS's supported
  ECMAScript and async behavior after Node adapters are removed?
- Can Pi's transaction callback shape be bridged without problematic nested
  asyncified QuickJS entry, or should the bridge accept a batch transaction
  representation?
- Which initial formatter and checker modules have acceptable bundle size,
  startup time, license, and WASI behavior?
- Which browser/version floor provides reliable OPFS and cross-origin isolation
  for the selected SQLite configuration?
- Does the product require direct access to user-selected directories later,
  and if so, how should that weaker portability and permission model coexist
  with the canonical SQLite project?
- Should model requests be proxied in all production deployments, or can a
  provider-specific OAuth flow avoid bearer keys in the browser host?

The first two questions are design-validation gates. A negative result requires
an explicit design revision; the implementation must not silently switch Pi to
native JavaScript while still describing the system as Pi-on-Wasm.

## References

- [pi-durable](https://github.com/earendil-works/pi/blob/main/packages/durable/README.md)
- [quickjs-emscripten](https://github.com/justjake/quickjs-emscripten)
- [SQLite Wasm](https://github.com/sqlite/sqlite-wasm)
- [SQLite Wasm persistence](https://www.sqlite.org/wasm/doc/trunk/persistence.md)
- [MDN: Origin private file system](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system)
- [Wasmer JavaScript SDK](https://docs.wasmer.io/runtime/js/)

