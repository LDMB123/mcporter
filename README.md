# MCPorter

TypeScript runtime, CLI, and code-generation toolkit for the Model Context
Protocol. MCPorter discovers MCP servers already configured on your machine,
calls their tools from a stable CLI or TypeScript API, and can generate
server-specific CLIs or typed clients when a workflow needs a smaller surface.

## Try It

```bash
npx mcporter list
npx mcporter list linear --schema
npx mcporter call linear.create_comment issueId:ENG-123 body:'Looks good!'
npx mcporter generate-cli linear --bundle dist/linear.js
npx mcporter emit-ts linear --mode client --out src/linear-client.ts
```

Human progress and prompts go to stderr. Use command-specific machine output
flags for scripts, such as `mcporter call --output json`.

## What It Does

- Discovers home, project, and imported MCP config from Cursor, Claude,
  Codex, Windsurf, OpenCode, and VS Code.
- Calls HTTP, SSE, and stdio MCP tools through one CLI/runtime surface.
- Handles OAuth setup and cached token refresh without putting secrets in
  project files.
- Keeps selected stateful servers warm through the daemon and can expose them
  through `mcporter serve`.
- Generates standalone CLIs with embedded schemas and typed clients for agent
  or test workflows.
- Records and replays MCP JSON-RPC traffic for offline debugging and redacted
  repros.

## Docs

- [Docs home](https://github.com/openclaw/mcporter/blob/main/docs/index.md)
- [Install](https://github.com/openclaw/mcporter/blob/main/docs/install.md)
- [Quickstart](https://github.com/openclaw/mcporter/blob/main/docs/quickstart.md)
- [Configuration](https://github.com/openclaw/mcporter/blob/main/docs/config.md)
- [CLI reference](https://github.com/openclaw/mcporter/blob/main/docs/cli-reference.md)
- [Ad-hoc connections](https://github.com/openclaw/mcporter/blob/main/docs/adhoc.md)
- [Tool calling](https://github.com/openclaw/mcporter/blob/main/docs/tool-calling.md)
- [Call syntax](https://github.com/openclaw/mcporter/blob/main/docs/call-syntax.md)
- [Generated CLIs](https://github.com/openclaw/mcporter/blob/main/docs/cli-generator.md)
- [Typed clients](https://github.com/openclaw/mcporter/blob/main/docs/emit-ts.md)
- [Agent skills](https://github.com/openclaw/mcporter/blob/main/docs/agent-skills.md)
- [Daemon](https://github.com/openclaw/mcporter/blob/main/docs/daemon.md)
- [Record/replay](https://github.com/openclaw/mcporter/blob/main/docs/record-replay.md)
- [Manual testing](https://github.com/openclaw/mcporter/blob/main/docs/manual-testing.md)
- [Release](https://github.com/openclaw/mcporter/blob/main/docs/RELEASE.md)

## Developer Workflow

Use pnpm and the repo runner so guardrails apply consistently:

```bash
pnpm install
./runner pnpm check
./runner pnpm test
./runner pnpm build
```

Useful local smokes:

```bash
./runner pnpm mcporter:list
./runner pnpm mcporter:call -- --help
```

Live MCP tests are opt-in:

```bash
MCP_LIVE_TESTS=1 ./runner pnpm test:live
```

## Safety

Keep OAuth tokens, bearer tokens, `.env*`, and provider credentials in local
environment, the MCPorter vault, or provider-managed stores. Project config may
reference environment variables, but should not store secret values directly.

## License

MIT. See [LICENSE](LICENSE).
