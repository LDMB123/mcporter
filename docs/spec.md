---
summary: 'Architecture overview for the mcporter runtime, CLI, generated-CLI toolkit, and typed-client layer.'
---

# mcporter Architecture

> Inspired in part by Anthropic’s guidance on MCP code execution agents: https://www.anthropic.com/engineering/code-execution-with-mcp

mcporter is a TypeScript runtime and CLI for calling Model Context Protocol servers from terminals, scripts, generated CLIs, and agents. For day-to-day command usage, see [CLI reference](./cli-reference.md); for install and first-run flows, see [Quickstart](./quickstart.md).

## Runtime Surface

- `createRuntime()` hosts shared connections, lists tools, calls tools, and resolves resources.
- `callOnce()` keeps one-shot scripts simple when connection reuse is unnecessary.
- `createServerProxy()` maps tool names to method-style calls, fills JSON-schema defaults, validates required arguments, and returns `CallResult` helpers for `.text()`, `.markdown()`, `.json()`, `.images()`, and `.content()`.
- The CLI (`npx mcporter list|call|auth|resource|serve|emit-ts|generate-cli`) uses the same runtime as the library API.

## Architecture Notes

- Load MCP definitions from JSON (support relative paths + HTTPS).
- Reuse `@modelcontextprotocol/sdk` transports; invoke stdio servers directly (e.g., call `npx` with env overrides) without an extra wrapper script.
- Automatically detect OAuth requirements for ad-hoc HTTP servers by retrying failed handshakes and promoting the definition to `auth: "oauth"` when a 401/403 is encountered, then launching the browser flow immediately.
- Mirror Python helper interpolation behavior for `${VAR}`, `${VAR:-default}`, and `$env:VAR`.
- Support OAuth token cache directory handling through `~/.mcporter`, XDG paths, and per-server overrides.
- Fetch tool signatures and schemas for `list`, generated CLIs, and typed clients.
- Provide lazy connection pooling per server to minimize startup cost.
- Document Cursor-compatible `config/mcporter.json` structure; support env-sourced headers and stdio commands while keeping inline overrides available for scripts.

## Schema-Aware Proxy Strategy

- Cache tool schemas on first access, persist them under `~/.mcporter/<server>/schema.json` or `$XDG_CACHE_HOME/mcporter/<server>/schema.json` for reuse across processes, and tolerate failures by falling back to raw `callTool`.
- Allow direct method-style invocations such as `context7.getLibraryDocs("react")` by:
  - Mapping camelCase properties to kebab-case tool names.
  - Detecting positional arguments and assigning them to required schema fields in order.
  - Handling multi-argument tools (e.g., Firecrawl’s `scrape`/`map`) via positional arrays, plain objects, or mixed option bags.
  - Merging JSON-schema defaults and validating required fields before dispatch.
- Return `CallResult` objects exposing `.raw`, `.text()`, `.markdown()`, `.json()` helpers for consistent post-processing.
- Keep implementation generic—no hardcoded knowledge of specific servers—so new MCP servers automatically gain the ergonomic API.
- Encourage lightweight composition helpers in examples (e.g., resolving then fetching Context7 docs) while keeping library exports generic.
- Back the proxy with targeted unit tests that cover primitive-only calls, positional tuples + option bags, and error fallbacks when schemas are missing.

## Standalone CLI Generation

- `generate-cli` should accept inline JSON, file paths, inline stdio commands (either via `--command` or as the first positional argument), or existing config names and produce a ready-to-run CLI that maps tools to Commander subcommands.
- Embed schemas (via `listTools { includeSchema: true }`) directly in the generated source so repeat executions avoid additional metadata calls.
- Support optional bundling through Rolldown by default, or Bun's native bundler
  when targeting Bun, with executable shebangs on bundled artifacts.
- Surface flags for output path, runtime target (`node` or `bun`), bundle destination, and per-call timeout (default 30s).
- See [CLI generator](./cli-generator.md) for current flags, bundling behavior, artifact metadata, and regeneration flows.

## Configuration

- Single file `config/mcporter.json` mirrors Cursor/Claude schema: `mcpServers` map with entries containing `baseUrl` or `command`+`args`, optional `headers`, `env`, `description`, `auth`, `tokenCacheDir`, and convenience `bearerToken`/`bearerTokenEnv` fields.
- Optional `imports` array (defaulting to ['cursor', 'claude-code', 'claude-desktop', 'codex', 'windsurf', 'vscode']) controls auto-merging of editor configs; entries earlier in the list win conflicts while local definitions can still override.
- Provide `configPath` override for scripts/tests; keep inline overrides in examples for completeness but default to file-based configuration.
- Add fixtures validating HTTP vs. stdio normalization, header/env behavior, and editor config imports (Cursor, Claude Code/Desktop, Codex, Windsurf, VS Code) to ensure priority ordering matches defaults.
