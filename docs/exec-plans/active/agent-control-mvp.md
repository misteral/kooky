# Agent control MVP

Turn kooky from a terminal that launches AI agents into a local control plane
for visible agent sessions. The MVP lets another local agent enumerate kooky
workspaces, inspect session context, read the visible terminal text, and send
input to a selected session.

## Why

The current app already has the hard part: `WorkspaceStore` owns all
workspaces, panes, sessions, cwd, agent identity, activity state, git/env
state, and per-session `KOOKY_SURFACE_ID` routing. What is missing is a stable
machine interface for an external agent to ask "what is open?" and "send this
to that session" without screen-scraping the UI or requiring tmux.

This should stay local-first:

- No network listener.
- No telemetry.
- No new Swift package dependencies.
- No required tmux dependency.
- No cloud/session sync.

## Decision

Add a separate local control socket plus a small CLI:

- `ControlServer` in the app listens on `~/Library/Application Support/kooky/control.sock`.
- `kookyctl` talks to that socket and prints text or JSON.
- The existing `HookServer` remains one-way agent telemetry only.
- The existing `TerminalEngine` protocol grows just enough read/write surface
  to support the control API.

Keep `HookServer` and `ControlServer` separate. Hook delivery is deliberately
small, fire-and-forget, and retry-sensitive. Control requests need replies,
larger payloads, and clearer error reporting; mixing them would make the hook
path easier to break.

## MVP scope

### Commands

```sh
kookyctl list --json
kookyctl snapshot --json
kookyctl capture --session <uuid>
kookyctl send --session <uuid> --text "npm test" --enter
kookyctl spawn --cwd /path/to/project --agent claude-code
```

### `list`

Returns lightweight topology:

- workspace id, title, cwd, active flag
- pane id, active flag
- session id, title, agent id/title, cwd, activity state
- git branch/dirty summary when known
- last command exit/duration when known

### `snapshot`

Returns the same topology as `list`, plus a compact per-session context block:

- cwd
- agent
- activity state
- branch and dirty counts
- environment indicators: Python venv, Node version, proxy name
- last command exit/duration
- visible terminal text for active sessions only

Do not include full scrollback in MVP. Visible text is enough for "what is this
agent doing now?" and avoids a first release that depends on scrollback
semantics not yet modeled in `TerminalEngine`.

### `capture`

Reads visible text from one session. Use libghostty's
`ghostty_surface_read_text` with a screen/surface selection, exposed through:

```swift
func readVisibleText() -> String
```

on `TerminalEngine`. If libghostty cannot provide stable visible-region text,
fallback MVP is "selected text only" is not acceptable; the feature should
pause until visible read works.

### `send`

Sends committed text into a session:

- `--text` sends exactly that text.
- `--enter` appends `\r`.
- Default path uses `sendInput`, not paste, because this is command/control
  input rather than user clipboard semantics.
- Later versions can add `--paste` when bracketed paste behavior is needed.

### `spawn`

Creates a new tab in the active workspace by default:

- `--cwd` sets initial cwd.
- `--agent` accepts an `AgentTemplate.id`.
- `--workspace <uuid>` optionally targets a workspace.
- Response returns the new session id and workspace id.

Worktree creation is explicitly out of MVP. The caller can create a worktree
before calling `spawn`.

## Wire format

Use newline-delimited JSON request/response over a unix domain socket.

Request:

```json
{"id":"req-1","method":"capture","params":{"sessionId":"..."}}
```

Success:

```json
{"id":"req-1","ok":true,"result":{"text":"..."}}
```

Failure:

```json
{"id":"req-1","ok":false,"error":{"code":"notFound","message":"session not found"}}
```

Initial error codes:

- `badRequest`
- `unknownMethod`
- `notFound`
- `unsupported`
- `internalError`

Keep payloads bounded. The server should cap capture output and return
`truncated: true` when needed. Start with a 128 KiB response cap.

## Implementation plan

### Phase 1: model snapshots

Add codable DTOs that can be built from `WorkspaceStore` without touching UI:

- `ControlWorkspaceSnapshot`
- `ControlPaneSnapshot`
- `ControlSessionSnapshot`
- `ControlGitSnapshot`
- `ControlEnvironmentSnapshot`

Add `WorkspaceStore.controlSnapshot(includeVisibleTextForActiveSessions:)`.

Tests:

- Snapshot includes restored workspaces/panes/tabs.
- Snapshot marks active workspace/pane/session correctly.
- Snapshot includes agent, cwd, activity, git/env, last command fields.

### Phase 2: visible capture

Extend `TerminalEngine`:

```swift
func readVisibleText() -> String
```

Implement in `LibghosttyEngine` with `ghostty_surface_read_text`.
Implement in `TestEngine` with a mutable `visibleText` fixture.

Tests:

- `WorkspaceStore` returns capture text for a known session.
- Unknown session returns a typed control error.
- Empty terminal returns an empty string, not an error.

Manual check:

- Run kooky, open a shell, print multi-line output, verify `kookyctl capture`
  matches visible screen text closely enough for agent context.

### Phase 3: control server

Add `ControlServer` next to `HookServer`:

- Bind `~/Library/Application Support/kooky/control.sock`.
- Accept one JSON request per connection.
- Dispatch on `.main` because `WorkspaceStore` and engines are `@MainActor`.
- Write exactly one JSON response, then close.
- Remove socket on shutdown.

Do not reuse `HookServer.parseMessage`; control requests have different
semantics and need request ids, responses, and errors.

Tests:

- Request parser accepts valid requests.
- Request parser rejects malformed JSON.
- Dispatcher handles `list`, `snapshot`, `capture`, `send`, `spawn`.
- Response encoder preserves request id.

### Phase 4: `kookyctl`

Add a small executable target that does not link `KookyKit`, mirroring
`KookyHook`'s low-dependency shape:

- Parse CLI args manually.
- Connect to `control.sock`.
- Print JSON for `--json`.
- Print plain text for `capture` by default.
- Exit `0` on success, `1` on server/control error, `2` on CLI usage error.

Commands for MVP:

```sh
kookyctl list [--json]
kookyctl snapshot [--json]
kookyctl capture --session <uuid>
kookyctl send --session <uuid> --text <text> [--enter]
kookyctl spawn --cwd <path> [--agent <id>] [--workspace <uuid>] [--json]
```

Packaging:

- Ensure `scripts/build-app.sh` includes `kookyctl` wherever `KookyHook` is
  already staged.
- Add the helper to the PATH directory kooky already prepends for agent shims,
  if that is the intended user-facing discovery path.

### Phase 5: agent-facing context contract

Document the expected external-agent workflow in `docs/ARCHITECTURE.md` after
the MVP works:

```sh
kookyctl snapshot --json
kookyctl capture --session <uuid>
kookyctl send --session <uuid> --text "..." --enter
```

This is documentation after implementation, not before. The active exec-plan is
the working spec until behavior is proven.

## Non-goals

- No tmux backend in MVP.
- No remote control over TCP/HTTP.
- No authentication beyond local user unix socket permissions.
- No full scrollback API.
- No waiting/prompt detection.
- No mouse/key event API.
- No automatic git worktree creation.
- No multi-agent scheduler.
- No UI for control permissions.

## Open questions

- Does `ghostty_surface_read_text` expose exactly the visible viewport, or does
  it require explicit row/column bounds derived from surface dimensions?
- Should `spawn` default to the active pane or active workspace's first pane?
  MVP should use active workspace + active pane because that matches user
  expectation from `Cmd+T`.
- Should `snapshot` include visible text for all sessions or only active
  sessions? MVP says active sessions only to control payload size.
- Should `send --enter` append `\r` or `\n`? Existing terminal input paths use
  carriage return for command submission; keep that convention unless testing
  proves otherwise.

## Definition of done

- `swift test` passes.
- `kookyctl list --json` shows all open workspaces/sessions in a running app.
- `kookyctl capture --session <uuid>` returns visible text from the target tab.
- `kookyctl send --session <uuid> --text "pwd" --enter` submits into the target
  tab without changing focus.
- `kookyctl spawn --cwd /tmp --agent terminal --json` creates a new session and
  returns its id.
- Killing/restarting kooky removes/recreates `control.sock` cleanly.
- Existing hook behavior and agent activity dots still work.

## Risks

### Capture fidelity

The MVP depends on `ghostty_surface_read_text`. If that API cannot produce a
stable visible-screen capture, do not fake it via selection or clipboard. Stop
and either expose the needed data through the libghostty bridge or defer
capture from MVP.

### Input safety

`send` is powerful: it can run commands in any visible session. Keep it local
socket only and do not add a network listener. Future permission UI can wait
until there is evidence users need it.

### Payload growth

External agents will ask for "all context". Keep `snapshot` compact and bounded
from the first version. Add scrollback/tails later as explicit commands, not as
default snapshot fields.

## Later versions

- `kookyctl wait --session <uuid> --state idle|attention --timeout 60`
- `kookyctl interrupt --session <uuid>`
- `kookyctl close --session <uuid>`
- `kookyctl rename --workspace/--session`
- `kookyctl worktree spawn --repo <path> --branch <name> --agent <id>`
- Scrollback capture with explicit line limits.
- Session labels/tasks so an external orchestrator can attach intent to tabs.
- Optional term-cli adapter for headless sessions.
- Optional tmux backend profile for remote or long-running detached work.
