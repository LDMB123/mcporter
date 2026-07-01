---
summary: 'Reference for the persistent MCP daemon used to keep long-lived servers (e.g., chrome-devtools) alive.'
read_when:
  - 'Implementing daemon/keep-alive behavior or maintaining its CLI commands.'
---

# MCPorter Daemon

The daemon keeps selected stateful MCP servers warm across separate CLI invocations. It is invisible for ordinary `mcporter call` and `mcporter list` flows unless a server opts into keep-alive behavior.

## Behavior

- **Invisible keep-alive:** `mcporter call` transparently starts (and reuses) a per-login daemon whenever a configured server requires persistence (e.g., `chrome-devtools` or CloudBase device authentication). No extra flags for agents.
- **Shared state:** Multiple CLI invocations/agents within the same user session reuse the same warm transport so STDIO servers can hold tabs, cookies, and other stateful context.
- **Per-config scope:** The daemon lives under the current user account, keyed by config path (`~/.mcporter/daemon/daemon-<config-hash>.sock` on Unix-like systems, or `$XDG_STATE_HOME/mcporter/daemon/...` when set), and never crosses user boundaries.
- **Resilience:** If the daemon or a keep-alive server crashes, the next CLI call restarts it automatically.
- **Explicit shutdown:** `mcporter daemon stop` tears everything down, and `mcporter daemon status` reports runtime state for debugging.
- **Configurable participation:** Only servers marked keep-alive participate; others keep ephemeral behavior. Config/env controls and the default allowlist govern opt-in/out.

## Architecture

- **Daemon process (`mcporter daemon start`):**
  - Loads the same config as the CLI.
  - Hosts a long-lived `McpRuntime`.
  - Listens on a Unix domain socket (per-login path, chmod 600).
  - Exposes a minimal JSON-RPC interface that mirrors the existing `list/call/resources` APIs so CLI commands can proxy requests.
  - Lazily connects keep-alive servers on first use and keeps transports open until shutdown or idle timeout.

- **Client shim (CLI side):**
  - When a command targets a keep-alive server:
    1. Look for a ready daemon socket; if missing, spawn `mcporter daemon start --detach`.
    2. Proxy the list/call/auth request over the socket and print the response as usual.
    3. If the socket handshake fails (daemon crashed mid-call), re-spawn once before surfacing the error.
  - Non keep-alive servers continue using the local runtime in the current process.

- **Keep-alive detection:**
  - Server definitions can set `lifecycle?: "ephemeral" | { mode: "keep-alive", idleTimeoutMs?: number }`.
  - Config-level defaults and `MCPORTER_KEEPALIVE` provide quick overrides.
  - The built-in allowlist (`chrome-devtools`, `mobile-mcp`, `playwright`, `cloudbase`) lets common stateful servers benefit immediately; users can opt out per server.

## CLI Surface

- `mcporter daemon start [--foreground]`: boot the daemon; default behavior is background (detached) launch that writes its metadata file under the daemon runtime directory.
- `mcporter daemon status`: show whether the daemon is running, the socket path, uptime, and which servers are currently connected/idle.
- `mcporter daemon stop`: instruct the daemon to close all transports and remove its socket/metadata; if the daemon is missing, exit 0 with a hint.
- `mcporter daemon restart`: convenience wrapper that stops the daemon (if it exists), waits for the socket to disappear, and launches a fresh instance while reusing the same logging flags/env overrides.
- Existing commands (`list`, `call`, `auth`, `emit-ts`, etc.) continue to work; only those touching keep-alive servers will route through the daemon.

## Lifecycle & Fault Handling

- **Auto start:** First call requiring the daemon triggers a lightweight bootstrap (fork/exec via `child_process.spawn` inside the CLI). We ensure the original command waits for the socket to become available (with a short timeout).
- **macOS Bun binaries:** Homebrew/Bun-compiled binaries wrap the detached child launch with `nohup` so the background daemon survives the parent CLI exit on macOS 26.
- **Auto restart:** The client shim treats `ECONNREFUSED`/broken pipe as a signal that the daemon died. It retries once by re-launching the daemon before surfacing the error.
- **Idle timeout:** Each keep-alive server can specify `idleTimeoutMs` (default `null` = never). The daemon tracks last activity timestamps and auto-closes transports (and associated external processes) after the idle window. A top-level config `daemonIdleTimeoutMs` can shut down the entire daemon after long inactivity.
- **Logging:** Daemon writes structured logs under the daemon runtime directory plus per-server logs for STDIO stderr so users can debug crashing servers.

## Agent Isolation

By default, multiple agents using the same config path share the same keep-alive daemon. That is deliberate: stateful servers such as browser or device MCPs can keep tabs, sessions, and subprocesses warm across repeated CLI calls.

If each agent needs independent MCP state, give each agent either:

- a distinct `--config <path>` / `MCPORTER_CONFIG` value, which produces a distinct daemon socket and metadata file; or
- a distinct `MCPORTER_DAEMON_DIR`, which isolates the whole daemon runtime directory even when the config path is shared. This explicit override wins over `XDG_STATE_HOME`.

Non-keep-alive servers remain process-local and do not use the daemon.

## Logging & Diagnostics

You can capture the daemon’s stdout/stderr (and per-server call traces) when debugging long-lived STDIO servers:

- `mcporter daemon start --log` enables logging with the default path `~/.mcporter/daemon/daemon-<config-hash>.log`, or `$XDG_STATE_HOME/mcporter/daemon/daemon-<config-hash>.log` when `XDG_STATE_HOME` is set. Use `--log-file <path>` to override it.
- `--log-servers chrome-devtools,mobile-mcp` restricts per-call logging to the listed servers. Without it, `--log` records every keep-alive server’s activity.
- Environment equivalents:
  - `MCPORTER_DAEMON_LOG=1` – enable logging.
  - `MCPORTER_DAEMON_LOG_PATH=/tmp/mcporter-daemon.log` – explicit log file.
  - `MCPORTER_DAEMON_LOG_SERVERS=chrome-devtools` – only log specified servers.
- `mcporter daemon status` now prints the socket path and the active log file (if any) so it’s easy to tail.
- Per-server opt-in: add `"logging": { "daemon": { "enabled": true } }` next to `"lifecycle": "keep-alive"` in a server definition to force detailed call logging for that server (handy when only one or two STDIO transports are noisy). Combined with `--log`/`MCPORTER_DAEMON_LOG`, those entries always emit call start/end/error lines.

Logs include timestamped entries such as:

```
[daemon] 2025-11-10T15:08:21.123Z callTool start server=chrome-devtools tool=take_snapshot
[daemon] 2025-11-10T15:08:22.004Z callTool success server=chrome-devtools tool=take_snapshot
```

Tailing the file (`tail -f ~/.mcporter/daemon/daemon-*.log`, or the matching XDG state path) surfaces crashes or repeated failures without needing to re-run the daemon in the foreground.

Agents can use persistent MCP servers without juggling multiple Chrome launches, while still retaining an explicit shutdown path.
