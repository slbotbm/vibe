# Glossary

These definitions use the meanings relevant to this proposed application.

| Term | Plain-language meaning | Read more |
| --- | --- | --- |
| Adapter | Code translating one component's interface into another's. | [Whole system](01-the-whole-system.md) |
| Agent | A program coordinating model calls and actions toward a task. | [Whole system](01-the-whole-system.md) |
| API | An agreed interface through which programs request work. | [Model transport](07-model-transport-and-streaming.md) |
| Async / await | JavaScript syntax for working with results that arrive later. | [Workers](02-javascript-and-workers.md) |
| Asyncify | A compile-time transformation allowing supported Wasm execution paths to suspend and resume. | [QuickJS](03-webassembly-and-quickjs.md) |
| Atomic | Committed as one whole operation, with no partial result. | [Storage](05-storage-and-project-files.md) |
| Backpressure | Slowing input consumption when the next stage cannot keep up. | [Model transport](07-model-transport-and-streaming.md) |
| Binding | A value supplied separately to a parameterized SQL statement. | [Storage](05-storage-and-project-files.md) |
| BLOB | Database value containing arbitrary bytes. | [Storage](05-storage-and-project-files.md) |
| BroadcastChannel | Browser API for messages between participating contexts on the same origin, subject to storage partitioning. | [Security](09-security-and-deployment.md) |
| Bundle | Build output combining application modules and dependencies. | [Workers](02-javascript-and-workers.md) |
| BYOK | Bring your own key: supplying your own model-provider credential. | [Model transport](07-model-transport-and-streaming.md) |
| Capability | A specific authority granted to code. | [Host calls](04-host-calls-and-capabilities.md) |
| Canonical store | The authoritative saved state from which views and copies are derived. | [Storage](05-storage-and-project-files.md) |
| Checkpoint | Saved task state used to reconstruct progress after restart. | [Recovery](06-durable-agents-and-recovery.md) |
| Commit / rollback | Make a transaction's changes durable / discard its uncommitted changes. | [Storage](05-storage-and-project-files.md) |
| COOP / COEP | Browser response policies used in establishing cross-origin isolation. | [Security](09-security-and-deployment.md) |
| CORS | Server rules controlling cross-origin access to responses by browser code. | [Security](09-security-and-deployment.md) |
| CPU / fuel | Processor execution / a runtime's optional measure for limiting execution work. | [Commands](08-wasi-and-command-execution.md) |
| CSP | Content Security Policy: browser restrictions on resources and execution. | [Security](09-security-and-deployment.md) |
| Delta / diff | Proposed data changes / a representation showing differences. | [Commands](08-wasi-and-command-execution.md) |
| DOM | The browser's object representation of page elements. | [Workers](02-javascript-and-workers.md) |
| Durable | Saved so committed progress can survive process restart, subject to storage guarantees. | [Recovery](06-durable-agents-and-recovery.md) |
| Effect ledger | Saved operation identities and outcomes used to prevent duplicate committed effects. | [Recovery](06-durable-agents-and-recovery.md) |
| Emscripten | A toolchain for compiling C/C++ programs for environments including browser Wasm. | [QuickJS](03-webassembly-and-quickjs.md) |
| Ephemeral | Temporary; not relied on to survive restart. | [Recovery](06-durable-agents-and-recovery.md) |
| FIFO | First in, first out: a queue's processing order. | [Workers](02-javascript-and-workers.md) |
| Harness | Agent coordination machinery surrounding the model. | [Recovery](06-durable-agents-and-recovery.md) |
| Hash | A fingerprint computed from bytes; used to compare arguments or verify assets. | [Storage](05-storage-and-project-files.md) |
| Host / guest | The environment supplying authority / code running inside it. | [Host calls](04-host-calls-and-capabilities.md) |
| Idempotent | Repetition produces the same effect as one execution. | [Recovery](06-durable-agents-and-recovery.md) |
| Immutable | Once created, a record is not modified; new records describe later events. | [Recovery](06-durable-agents-and-recovery.md) |
| IndexedDB / JSONL | Browser object database / line-separated JSON record format; alternative persistence approaches discussed in the design. | [Storage](05-storage-and-project-files.md) |
| Integrity | Assurance that bytes match the trusted expected contents. | [Security](09-security-and-deployment.md) |
| JSON | A text format for ordinary structured data. | [Host calls](04-host-calls-and-capabilities.md) |
| Linear memory | Byte buffer used by Wasm's memory instructions. | [QuickJS](03-webassembly-and-quickjs.md) |
| Local-first | The browser owns project and agent state; remote services may still be used. | [Whole system](01-the-whole-system.md) |
| Mailbox | Queue through which work enters a runtime in a controlled order. | [Workers](02-javascript-and-workers.md) |
| Migration | A versioned change to stored database structure or data. | [Storage](05-storage-and-project-files.md) |
| Module / instance | A Wasm code definition / a live execution with state and imports. | [QuickJS](03-webassembly-and-quickjs.md) |
| Namespace | Naming convention separating groups of objects, such as `pi_` and `app_` tables; not security by itself. | [Host calls](04-host-calls-and-capabilities.md) |
| NFC | A Unicode normalization form ensuring canonically equivalent filename text uses a consistent representation. | [Storage](05-storage-and-project-files.md) |
| Node built-in | Facility supplied by Node.js rather than the JavaScript language itself. | [Workers](02-javascript-and-workers.md) |
| OAuth | An authorization protocol family; mentioned as a possible future credential approach, not selected here. | [Model transport](07-model-transport-and-streaming.md) |
| OPFS | Origin private file system: browser-managed persistent file storage. | [Storage](05-storage-and-project-files.md) |
| Origin | URL scheme, hostname, and port identifying a browser security boundary. | [Storage](05-storage-and-project-files.md) |
| POSIX | Operating-system interface standards; not supplied wholesale by this application. | [Commands](08-wasi-and-command-execution.md) |
| Preopened directory | A directory capability explicitly supplied to a command runner. | [Commands](08-wasi-and-command-execution.md) |
| Promise | A JavaScript representation of a result that may arrive later. | [Workers](02-javascript-and-workers.md) |
| Protocol | The agreed message formats and rules between components. | [Host calls](04-host-calls-and-capabilities.md) |
| QuickJS | A JavaScript engine used here as a Wasm-hosted guest runtime. | [QuickJS](03-webassembly-and-quickjs.md) |
| Relay | A server forwarding model traffic and attaching credentials. | [Model transport](07-model-transport-and-streaming.md) |
| Revision | Version number used to identify saved state and detect stale edits. | [Storage](05-storage-and-project-files.md) |
| Runtime | The environment executing code and supplying its facilities. | [Workers](02-javascript-and-workers.md) |
| Sandbox | Restricted environment with limited access to surrounding resources. | [Commands](08-wasi-and-command-execution.md) |
| Schema | Definition of valid data fields or database structure. | [Host calls](04-host-calls-and-capabilities.md) |
| Service worker | Browser-managed event worker used here for asset caching, with a limited lifecycle. | [Security](09-security-and-deployment.md) |
| Snapshot | A coherent view of state at a particular point or revision. | [Commands](08-wasi-and-command-execution.md) |
| SQLite | Embedded SQL database engine; SQLite-Wasm is a Wasm build. | [Storage](05-storage-and-project-files.md) |
| SSE | Server-sent events: a text framing format for incremental server messages. | [Model transport](07-model-transport-and-streaming.md) |
| State machine | Named states plus rules for moving between them. | [Recovery](06-durable-agents-and-recovery.md) |
| stdout / stderr | A command's ordinary output / diagnostic output streams. | [Commands](08-wasi-and-command-execution.md) |
| Telemetry | Measurements and operational events used to understand behavior. | [Security](09-security-and-deployment.md) |
| Transaction | Group of database changes committed or rolled back together. | [Storage](05-storage-and-project-files.md) |
| UUID | A standardized identifier format, used for distinct requests and records. | [Host calls](04-host-calls-and-capabilities.md) |
| VFS | Virtual filesystem interface connecting storage operations to backing bytes. | [Storage](05-storage-and-project-files.md) |
| VM | Virtual machine: a software execution environment; QuickJS is not a full guest operating system. | [QuickJS](03-webassembly-and-quickjs.md) |
| WASI | Interfaces a host can supply to compatible Wasm programs. | [Commands](08-wasi-and-command-execution.md) |
| Wasm | WebAssembly: a portable compiled instruction format. | [QuickJS](03-webassembly-and-quickjs.md) |
| Web Lock | Browser coordination primitive used to grant one project writer ownership. | [Security](09-security-and-deployment.md) |
| WebContainer | A browser-based Node-style runtime considered as an alternative in the design. | [Whole system](01-the-whole-system.md) |
| Web Worker | A browser execution environment separate from the page's main thread. | [Workers](02-javascript-and-workers.md) |

Some names in the design, such as Pi AI and Chord, identify Pi's dependencies.
Their internals are not prerequisites for understanding the architecture: start
with the [harness and recovery chapter](06-durable-agents-and-recovery.md), then
consult the pinned packages' documentation when implementing their adapters.
