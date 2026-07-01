---
summary: 'Reference for mcporter generate-cli outputs, runtimes, schema-aware UX, and artifact regeneration.'
read_when:
  - 'Changing generate-cli behavior or bundler integrations'
---

# CLI Generator

Default behavior: generating `<server>.ts` in the working directory if no output path is provided. Bundling is opt-in via `--bundle` and produces a single JS file with shebang; otherwise we emit TypeScript targeting Node.js. Rolldown handles bundling by default unless the runtime resolves to Bun—in that case Bun’s native bundler is selected automatically (still requires `--runtime bun` or Bun auto-detection); `--bundler` lets you override either choice.

## Overview

`mcporter generate-cli` produces a standalone CLI for a single MCP server. The generated CLI feels like a Unix tool: subcommands map to MCP tools, arguments translate to schema fields, and output can be piped or redirected.

## Behavior

- **Input**: Identify the target server either by shorthand name or by providing an explicit MCP server definition.
- **Output**: Emit a TypeScript file (ESM) targeting Node.js by default (`<server>.ts` unless `--output` overrides). Bundling to a standalone JS file happens only when `--bundle` is passed.
- **Runtime Selection**: Prefer Bun when it is available (`bun --version` succeeds); otherwise fall back to Node.js. Callers can force either runtime via `--runtime bun|node`.
- **Schema-Aware CLI**: Leverage `createServerProxy` to map positional/flag arguments to MCP tool schemas, including defaults and required validation.
- **Unix-Friendly Output**: Provide `--output text|json|markdown|raw` flags so results can be piped; default to human-readable text. Include `--timeout` (default 30s) to cap call duration.

## Notes

- Generated CLI depends on the latest `commander` for argument parsing.
- Default timeout for tool calls is 30 seconds, overridable via `--timeout`.
- Runtime flag remains (`--runtime bun`) to tailor shebang/usage instructions, but Node.js is the default.
- Generated CLI embeds the resolved server definition and always targets that snapshot (no external `--config` or `--server` overrides at runtime).

## Usage Examples

```bash
# Minimal: infer the name from the command URL and emit TypeScript (optionally bundle)
npx mcporter generate-cli \
  --command https://mcp.context7.com/mcp \
  --minify

# Provide explicit name/description and compile a Bun binary (falls back to Node if Bun missing)
npx mcporter generate-cli \
  --name context7 \
  --command https://mcp.context7.com/mcp \
  --description "Context7 docs MCP" \
  --runtime bun \
  --compile

chmod +x context7
./context7
  # show the embedded help + tool list

# Shareable "one weird trick" for chrome-devtools (no config required)
npx mcporter generate-cli --command "npx -y chrome-devtools-mcp@latest"
```

- `--minify` shrinks the bundled output via the selected bundler (output defaults to `<server>.js`).
- `--compile [path]` implies bundling and invokes `bun build --compile` to create the native executable (Bun only). When you omit the path, the compiled binary inherits the server name.
- Use `--server '{...}'` when you need advanced configuration (headers, env vars, stdio commands, OAuth metadata).
- Omit `--name` to let mcporter infer it from the command URL (for example, `https://mcp.context7.com/mcp` becomes `context7`).
- When targeting an existing config entry, you can skip `--server` and pass the name as a positional argument:
  `npx mcporter generate-cli linear --bundle dist/linear.js`.
- When the MCP server is a stdio command, you can also skip `--command` by quoting the inline command as the first positional argument (e.g., `npx mcporter generate-cli "npx -y chrome-devtools-mcp@latest"`).
- Generated CLIs preserve `lifecycle: "keep-alive"` for embedded stdio servers. At runtime they create a stable generated config under `~/.mcporter/generated/` (or `$XDG_STATE_HOME/mcporter/generated/` when set), auto-start the daemon as needed, and keep the server process alive across separate generated-CLI invocations.
- Narrow the CLI to a specific subset of tools with `--include-tools`:
  `npx mcporter generate-cli linear --include-tools issues_list,issues_create`.
- Hide debug or admin tools with `--exclude-tools`:
  `npx mcporter generate-cli linear --exclude-tools debug_tool,admin_reset`.

## Artifact Metadata & Regeneration

- Every generated artifact embeds its metadata (generator version, resolved server definition, invocation flags). A hidden `__mcporter_inspect` subcommand prints the payload without contacting the MCP server, so binaries remain self-describing even after being copied to another machine.
- `mcporter inspect-cli <artifact>` shells out to that embedded command and prints a human summary (pass `--json` for raw output). The summary includes a ready-to-run `generate-cli` command you can reuse directly.
- `mcporter generate-cli --from <artifact>` replays the stored invocation against the latest mcporter build. `--server`, `--runtime`, `--timeout`, `--minify/--no-minify`, `--bundle`, `--compile`, `--output`, and `--dry-run` let you override specific pieces of the stored metadata when necessary.
- Because the metadata lives inside the artifact, any template, bundle, or compiled binary can be refreshed after a generator upgrade without juggling sidecar files.
