---
title: Public Go Client Contract for Sandbox Lifecycle and Agent-Runtime Access
authors:
  - "@ljluestc"
reviewers:
  - TBD
creation-date: 2026-09-25
last-updated: 2026-09-25
status: provisional
see-also:
  - "/docs/proposals/20260713-traffic-access-token-jwt-verification.md"
  - "/docs/proposals/20260725-sandbox-ingress-authn.md"
---

# Public Go Client Contract for Sandbox Lifecycle and Agent-Runtime Access

## Table of Contents

- [Public Go Client Contract for Sandbox Lifecycle and Agent-Runtime Access](#public-go-client-contract-for-sandbox-lifecycle-and-agent-runtime-access)
  - [Table of Contents](#table-of-contents)
  - [Glossary](#glossary)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals/Future Work](#non-goalsfuture-work)
  - [Proposal](#proposal)
    - [User Stories](#user-stories)
    - [Requirements](#requirements)
    - [Client Shape](#client-shape)
    - [Operation Semantics](#operation-semantics)
    - [Security and Path Semantics](#security-and-path-semantics)
    - [Error Taxonomy](#error-taxonomy)
    - [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
    - [Risks and Mitigations](#risks-and-mitigations)
  - [Alternatives](#alternatives)
  - [Upgrade Strategy](#upgrade-strategy)
  - [Additional Details](#additional-details)
    - [Test Plan](#test-plan)
  - [Open Questions](#open-questions)
  - [Implementation History](#implementation-history)

## Glossary

- **Manager**: the sandbox-manager E2B-compatible HTTP API (`pkg/servers/e2b`),
  authenticated with `X-API-Key`.
- **Runtime**: the envd-compatible agent-runtime inside each sandbox, serving
  the HTTP `/files` route and the connect-rpc `filesystem.Filesystem` and
  `process.Process` services on `utils.RuntimePort` (49983).
- **Gateway**: the sandbox-gateway, which routes external traffic to a
  sandbox's runtime by sandbox ID and port.
- **Envd access token**: the per-sandbox token returned as `envdAccessToken`
  and sent to the runtime as `X-Access-Token`.
- **Traffic access token**: the gateway-scoped token returned as
  `trafficAccessToken`, refreshable through
  `POST /kruise/api/sandboxes/{sandboxID}/traffic-access-token`.

## Summary

Third-party Go agent frameworks, starting with
[AgentScope Go](https://github.com/agentscope-ai/agentscope-go)
([#998](https://github.com/openkruise/agents/issues/998)), want to use
OpenKruise Agents as an optional sandbox backend for their file and command
tools. The only in-repo Go client, `pkg/utils/runtime`, takes
`*v1alpha1.Sandbox` objects and reads Kubernetes annotations, so an
out-of-cluster application cannot use it without importing the CRD types and
controller dependencies.

This proposal defines a small, experimental, public Go client contract. The
contract is built only on wire protocols that OpenKruise already serves: the
Manager's E2B-compatible lifecycle routes and the Runtime's envd `/files` route
and connect-rpc services. It specifies the minimum operations that an agent
tool backend needs, namely create, delete, and connect for the lifecycle, and
execute, read, write, list, remove, and mkdir for the runtime. It also defines
their byte-fidelity, path, authentication, timeout, and error semantics. The
client will live in a separate Go module that has no dependency on `api/`,
`pkg/`, or Kubernetes libraries.

## Motivation

- AgentScope Go and similar frameworks expose a `Workspace`-style abstraction
  (write, read, list, remove, and execute) that their Bash, read, write, edit,
  grep, and glob tools run against. OpenKruise already provides isolation,
  pooling, and lifecycle, but it has no documented contract that a Go
  integrator can code against.
- The E2B SDKs are the de facto contract today, but there is no first-party
  Go SDK. The integrator would have to reverse-engineer which E2B behaviors
  OpenKruise supports, such as `GET /files` for reads, `username` resolution,
  and whether parent directories are created.
- `pkg/utils/runtime` is an internal control-plane helper. It is bound to
  `*v1alpha1.Sandbox`, resolves addresses from annotations and Pod IPs, and
  returns connect/protobuf types. Its semantics are designed for
  sandbox-manager's own use, such as CSI mounts and init. Those semantics must
  stay free to change.

### Goals

- Define a protocol-neutral Go contract for the MVP operations listed in #998.
- Pin down, for each operation, byte fidelity, parent-directory behavior,
  recursion, size limits, timeouts, the executing user, and error
  classification.
- Keep the contract dependency-free with respect to CRDs, controller-runtime,
  and internal packages, and make it testable against a fake HTTP server
  without a cluster.
- Reuse existing wire protocols. The MVP requires no new Manager route and no
  new runtime RPC.
- Ship with contract tests so that any runtime implementation can be checked
  against the documented semantics.

### Non-Goals/Future Work

- Pause, resume, snapshot, timeout extension, network updates, and warm-pool
  management. These are follow-ups that map directly onto existing Manager
  routes.
- Streaming execution, PTY, stdin, file watching, and `Move`/`Stat` in the
  public surface.
- Changing the E2B wire protocol or the envd protobufs.
- Replacing the internal `pkg/utils/runtime` client or migrating
  sandbox-manager onto the new module.
- Adding an AgentScope dependency to OpenKruise. The AgentScope-side adapter
  stays in AgentScope.

## Proposal

### User Stories

#### Story 1: Agent framework tool backend

An AgentScope Go application configures an OpenKruise manager URL and API key,
creates a sandbox from template `python-3.12`, and plugs the returned runtime
handle into its `Workspace` adapter. Its Bash tool calls `Execute`, and its
edit tool calls `ReadFile` followed by `WriteFile`. When the session ends, the
application calls `DeleteSandbox`.

#### Story 2: In-cluster caller

An in-cluster service that already knows a sandbox ID calls `Connect(ctx, id)`
and reaches the runtime through the gateway, with the same semantics as an
external caller.

#### Story 3: Offline testing

An integrator runs the published contract tests against
`clienttest.NewFakeRuntime()`, an `httptest.Server` that implements the
documented semantics. This lets the integrator verify their adapter in CI
without Kubernetes.

### Requirements

#### Functional Requirements

- **FR1** Configure the Manager base URL, API key, sandbox domain, HTTP client,
  and TLS settings (CA bundle and optional client certificate).
- **FR2** `CreateSandbox` from a template ID, with timeout, metadata, and env
  vars. Maps to `POST /sandboxes`.
- **FR3** `DeleteSandbox` maps to `DELETE /sandboxes/{id}`. It is idempotent on
  not-found only when the caller opts in.
- **FR4** `Connect` resolves the runtime endpoint and credentials for an
  existing sandbox. It maps to `POST /sandboxes/{id}/connect`, which also
  resumes a paused sandbox.
- **FR5** Runtime `Execute`, `ReadFile`, `WriteFile`, `ListDir`, `RemovePath`,
  and `MakeDir` behave as specified in
  [Operation Semantics](#operation-semantics).

#### Non-Functional Requirements

- **NFR1** The module imports only the standard library, `connectrpc.com/connect`,
  and `google.golang.org/protobuf`. It does not import `k8s.io/*`,
  `sigs.k8s.io/*`, `github.com/openkruise/agents/api`, or
  `github.com/openkruise/agents/pkg`.
- **NFR2** The Manager API key is never sent to the runtime or the gateway
  data plane.
- **NFR3** Every call honors `ctx` cancellation and has a bounded default
  timeout.
- **NFR4** All errors are classifiable with `errors.Is` against exported
  sentinels.

### Client Shape

```go
package kruiseagents // module github.com/openkruise/agents/sdk/go

type Config struct {
    ManagerURL string        // e.g. https://sandbox.example.com/kruise/api
    APIKey     string        // sent only to ManagerURL as X-API-Key
    Domain     string        // gateway domain for runtime traffic
    HTTPClient *http.Client  // optional; carries TLS/CA/client-cert config
}

func NewClient(cfg Config) (*Client, error)

func (c *Client) CreateSandbox(ctx context.Context, opts CreateSandboxOptions) (*Sandbox, error)
func (c *Client) Connect(ctx context.Context, sandboxID string) (*Sandbox, error)
func (c *Client) DeleteSandbox(ctx context.Context, sandboxID string) error

// Sandbox is a handle carrying the sandbox ID and the runtime credentials
// returned by the Manager. It never holds the Manager API key.
type Sandbox struct {
    ID         string
    TemplateID string
    // unexported: runtime base URL, envd access token, traffic access token
}

func (s *Sandbox) Runtime() *Runtime

func (r *Runtime) Execute(ctx context.Context, req ExecuteRequest) (*ExecuteResult, error)
func (r *Runtime) ReadFile(ctx context.Context, path string, opts ...FileOption) ([]byte, error)
func (r *Runtime) WriteFile(ctx context.Context, path string, data []byte, opts ...FileOption) error
func (r *Runtime) ListDir(ctx context.Context, dir string, opts ...FileOption) ([]DirEntry, error)
func (r *Runtime) RemovePath(ctx context.Context, path string, opts ...FileOption) error
func (r *Runtime) MakeDir(ctx context.Context, dir string, opts ...FileOption) error

type ExecuteRequest struct {
    Command string            // run as: /bin/bash -l -c <Command>
    Cwd     string            // optional
    Envs    map[string]string // optional
    User    string            // optional; default "user"
    Timeout time.Duration     // optional; default 60s
}

type ExecuteResult struct {
    Stdout   []byte
    Stderr   []byte
    ExitCode int
    // Truncated reports that Stdout or Stderr hit MaxOutputBytes.
    Truncated bool
}

type DirEntry struct {
    Name    string
    Path    string
    Type    EntryType // File | Dir | Symlink
    Size    int64
    Mode    fs.FileMode
    ModTime time.Time
}
```

The types are plain Go structs. No protobuf or connect types appear in the
public signatures, so the wire protocol can evolve behind the facade.

### Operation Semantics

| Operation | Wire mapping | Semantics |
|---|---|---|
| `CreateSandbox` | Manager `POST /sandboxes` | Returns the `Sandbox` handle with `envdAccessToken`, `trafficAccessToken`, and `domain`. |
| `Connect` | Manager `POST /sandboxes/{id}/connect` | Resumes a paused sandbox and returns fresh runtime credentials. |
| `DeleteSandbox` | Manager `DELETE /sandboxes/{id}` | A not-found response returns `ErrNotFound`. |
| `Execute` | Runtime `process.Process/Start` (server stream) | Runs to completion and collects stdout and stderr as raw bytes. A non-zero exit code is **not** an error; it is reported in `ExitCode`. A timeout returns `ErrTimeout`. Output is capped at `MaxOutputBytes` (default 10 MiB per stream), and the result sets `Truncated` when the cap is hit. |
| `ReadFile` | Runtime `GET /files?path=&username=` | Returns the raw body, byte for byte. Reads larger than `MaxReadBytes` (default 100 MiB) fail with `ErrTooLarge` and are not truncated silently. `ReadFileStream` returning an `io.ReadCloser` is a follow-up. |
| `WriteFile` | Runtime `POST /files?path=&username=` (multipart) | Is binary-safe and overwrites an existing file. **Creates missing parent directories.** The internal client documents that the parent must already exist, so the public client guarantees this behavior itself. It calls `MakeDir(parent)` before the upload unless the caller passes `WithoutParents()`. |
| `ListDir` | Runtime `filesystem.Filesystem/ListDir` (depth 1) | Non-recursive. Returns name, absolute path, type, size, mode, and mtime. Symlinks are reported, not followed. |
| `RemovePath` | Runtime `filesystem.Filesystem/Remove` | Removes files, and removes directories **recursively**. A missing path returns `ErrNotFound`. Callers may wrap it with `IgnoreNotFound`. |
| `MakeDir` | Runtime `filesystem.Filesystem/MakeDir` | Creates the directory and any missing parents (`mkdir -p`). An existing directory is not an error. |

The default per-call timeout is 30s for file operations and 60s for
`Execute`. Both can be overridden per call. Every call also honors the deadline
of `ctx`.

### Security and Path Semantics

- **Paths** are absolute paths inside the sandbox. A relative path is resolved
  against the executing user's home directory, which matches envd behavior.
  The contract does not add a workspace-root jail. Confinement to a workspace
  root stays the adapter's responsibility, as AgentScope already does. The
  sandbox is the isolation boundary.
- **Symlinks** are followed by read and write, as in envd. `ListDir` reports
  them as `Symlink` without following them.
- **User**: file operations and commands run as `username`. The default is
  `user`, which matches the E2B SDK default. Callers may pass `root`.
  Internally, sandbox-manager uses `root` for its own writes. The public client
  does not inherit that default.
- **Authentication**:
  - Manager calls send `X-API-Key` and nothing else.
  - Runtime calls send `X-Access-Token: <envdAccessToken>` for runtime
    authentication. When the sandbox is exposed through the gateway with
    traffic authentication, they also send the traffic access token in
    `e2b-traffic-access-token`, which is the gateway default and is
    configurable. The gateway strips this header before forwarding.
  - The API key is never attached to runtime requests. This is enforced in the
    client and covered by a contract test.
- **Token refresh**: the traffic access token has an expiration. `Runtime`
  refreshes it on demand through
  `POST /kruise/api/sandboxes/{id}/traffic-access-token` shortly before
  expiry, and once on an auth failure (`401`/`Unauthenticated`), before it
  returns `ErrUnauthenticated`.
- **TLS**: the caller supplies an `*http.Client`, which carries custom CA and
  client-certificate configuration. The client never disables verification.
- **Addressing**: the default runtime URL is
  `https://49983-<sandboxID>.<domain>` through the gateway. Direct Pod
  addressing is out of scope for the public contract, because it would
  reintroduce Kubernetes knowledge.

### Error Taxonomy

Exported sentinels are all matchable with `errors.Is`. Each error wraps the
underlying HTTP status or connect code for diagnostics.

| Sentinel | Manager HTTP | Runtime HTTP / connect code |
|---|---|---|
| `ErrUnauthenticated` | 401 | 401 / `Unauthenticated` |
| `ErrPermissionDenied` | 403 | 403 / `PermissionDenied` |
| `ErrNotFound` | 404 | 404 / `NotFound` |
| `ErrAlreadyExists` | 409 | `AlreadyExists` |
| `ErrTooLarge` | 413 | 413 / client-side cap |
| `ErrTimeout` | `ctx` deadline | `DeadlineExceeded` / `ctx` deadline |
| `ErrUnavailable` | 5xx, transport | 502/503 / `Unavailable`, transport |
| `ErrUnsupported` | 501 | `Unimplemented` (older runtime) |

A non-zero exit code from `Execute` is not an error.

### Implementation Details/Notes/Constraints

- **Location**: this proposal places the client in a new nested Go module,
  `sdk/go` (`github.com/openkruise/agents/sdk/go`), next to the existing
  Python `sdk/customized_e2b`. A separate module guarantees NFR1 through
  `go.mod` rather than through review. It also keeps sandbox-manager and the
  controller out of the dependency closure in both directions, which is
  consistent with the layering rules in `AGENTS.md`.
- **Protobufs**: `proto/envd` is generated code in the root module. The SDK
  module either vendors a copy of the two envd `.proto` packages or depends on
  a small, separately versioned `proto` module. Reviewers should decide
  between these options, as listed under [Open Questions](#open-questions).
- **No server change is required for the MVP**. Every operation maps to a
  route or RPC that is already served and already exercised by the E2B test
  suite (`test/e2b/test_filesystem.py` covers `files.read`/`files.write`).
- **Relationship to `pkg/utils/runtime`**: the internal client is unchanged.
  Sharing classification helpers is deliberately avoided, so the public
  module does not import internal packages.
- **Stability**: the module starts at `v0.x` and is documented as
  experimental. Breaking changes are announced in `CHANGELOG.md`.

### Risks and Mitigations

- **Divergence between envd and the documented semantics**, for example
  parent-directory creation or symlink handling. *Mitigation*: contract tests
  run against a real runtime in the existing E2E environment and against the
  fake server. The fake server encodes the documented behavior, not observed
  behavior.
- **Maintenance cost of a second client**. *Mitigation*: the MVP surface is
  deliberately small (nine methods) and wire-level only. Lifecycle follow-ups
  are thin mappings.
- **Token leakage**. *Mitigation*: the API key is scoped to the Manager
  transport by construction, with a contract test that asserts it never
  reaches the runtime. Tokens are redacted from errors and logs.
- **Unbounded memory on read or exec**. *Mitigation*: default caps with
  explicit `ErrTooLarge` or `Truncated`. Streaming variants are planned.

## Alternatives

1. **Tell integrators to use an E2B Go port.** No first-party E2B Go SDK
   exists, and E2B semantics that OpenKruise does not implement would leak
   into the contract. This alternative is rejected as the primary path. The
   wire protocol stays E2B-compatible, so a future E2B Go SDK would still
   work.
2. **Export `pkg/utils/runtime`.** It is bound to `*v1alpha1.Sandbox`,
   Kubernetes annotations, and Pod-IP addressing. Exporting it would freeze
   internal control-plane semantics and pull in the Kubernetes dependency
   closure. Rejected.
3. **Add a native `ReadFile` RPC to the envd Filesystem service.** `GET
   /files` already provides binary-safe reads, and changing the vendored
   upstream envd protobufs would fork them. The contract hides the transport,
   so a native RPC can be adopted later without API change. Deferred.
4. **Proxy file and command operations through new Manager routes such as
   `/sandboxes/{id}/files`.** This would put data-plane traffic on the control
   plane and double-hop every byte. Rejected.

## Upgrade Strategy

The change is purely additive. The feature introduces a new module and changes
no existing server, CRD, route, or flag. Runtimes that predate the Filesystem
service return `ErrUnsupported` from `ListDir`, `RemovePath`, and `MakeDir`,
while `ReadFile`, `WriteFile`, and `Execute` keep working.

## Additional Details

### Test Plan

- Unit tests in `sdk/go` against an `httptest.Server`-based fake Manager and
  fake Runtime, table-driven over the semantics table and the error
  taxonomy. The tests cover binary round-trip with all 256 byte values,
  parent-directory creation, recursive remove, non-zero exit code handling,
  timeouts, output caps, and the invariant that the API key is never sent to
  the runtime.
- A `clienttest` package that exports the fake servers, so that integrators
  such as the AgentScope adapter can reuse them.
- An optional E2E job that runs the same contract suite against a real
  sandbox in the existing E2E environment. It is not required for ordinary
  validation.

## Open Questions

1. Should the SDK live in `sdk/go` in this repository or in a separate
   repository, such as `openkruise/agents-go-sdk`?
2. How should the envd protobufs be shared: vendored into the SDK module, or
   published as a small `proto` module?
3. Should in-cluster callers be allowed to bypass the gateway with a direct
   runtime URL option? The current proposal answers no for the MVP.
4. Should `Stat` and `Move` enter the public surface in v0, or stay envd
   details until there is a demonstrated need?
5. Is the E2B default `username` of `user` the right default for all
   OpenKruise templates, or should templates advertise a default user?

## Implementation History

- [x] 09/2026: Proposed in [#998](https://github.com/openkruise/agents/issues/998)
- [x] 09/25/2026: Open proposal PR
- [ ] First round of feedback from maintainers and the AgentScope community
- [ ] Implement `sdk/go` MVP and contract tests
- [ ] AgentScope `OpenKruiseWorkspace` adapter validated against the contract tests
