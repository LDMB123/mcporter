---
summary: 'Living list of mcporter limitations, hosted MCP quirks, and upstream gaps.'
read_when:
  - 'Triaging a bug that might already be documented'
---

# Known Issues

This file tracks limitations that users regularly run into. Most of these require upstream cooperation or larger refactors—feel free to reference this when triaging bugs.

## Hosted OAuth servers (Supabase, GitHub MCP, etc.)

- Supabase’s hosted MCP server (`https://mcp.supabase.com/mcp`) rejects the standard `mcp:tools` scope and only accepts Supabase-specific scopes such as `organizations:read`, `projects:read`, `projects:write`, `database:read`, `database:write`, `analytics:read`, `secrets:read`, `edge_functions:read`, `edge_functions:write`, `environment:read`, `environment:write`, and `storage:read`. The OAuth failure usually appears as HTTP 400 with wording like `scope.0: Invalid enum value ... received 'mcp:tools'`. Because Supabase does not publish OAuth discovery metadata with `scopes_supported`, mcporter cannot negotiate the hosted flow automatically. Workarounds:
  - Use Supabase’s supported clients (Cursor, Windsurf).
  - Self-host their MCP server and configure PAT headers / custom OAuth.
  - Try a configured `oauthScope` only when you can supply the provider's exact
    supported scope string; this is provider-dependent and does not fix missing
    discovery metadata.
  - Ask Supabase to accept the MCP scope or publish their scope list.
- GitHub’s MCP endpoint (`https://api.githubcopilot.com/mcp/`) returns “does not support dynamic client registration” when mcporter attempts to connect. Copilot’s backend expects pre-registered client credentials. Configure `oauthClientId`/`oauthClientSecretEnv` only if the provider gives you a usable OAuth app; otherwise use their supported client or token/header workaround.
- Some hosted servers reject dynamic client registration before returning any authorization URL. mcporter now fails those flows immediately instead of waiting for a browser callback that cannot arrive. If the provider supports a pre-registered OAuth app, configure `oauthClientId`, `oauthClientSecretEnv`, and the required `oauthTokenEndpointAuthMethod`; otherwise use the provider's supported client or token/header workaround.
- `mcporter auth <server> --no-browser` still starts a loopback callback server and must stay alive until the browser redirects back. Process managers that run commands in short-lived process groups can print the authorization URL and then reap the process tree, leaving no listener on the callback port and no saved tokens. Run headless OAuth from a persistent terminal, `tmux`, or `nohup`/a supervisor, and use a configured `oauthRedirectUrl` or loopback tunnel when the browser runs elsewhere.

## Output schemas missing/buggy on many servers

- The MCP spec allows servers to omit `outputSchema`. In practice, many hosted MCPs return empty or inconsistent schemas, so features that rely on return types (TypeScript signatures, generated CLIs, `createServerProxy` return helpers) may degrade to `unknown`.
- Workarounds: inspect the server’s README / manual docs for output details, or wrap the tool via `createServerProxy` and handle the raw envelope manually.
- Potential improvement: allow user-provided schema overrides so we can fill gaps on a per-tool basis.

## MCP SDK 1.22.0 inline-stdio regression

- Upgrading `@modelcontextprotocol/sdk` to 1.22.0 causes `mcporter generate-cli --compile` (and direct runtime `listTools`) to fail against inline STDIO servers with `MCP error -32603: Cannot read properties of undefined (reading 'typeName')`.
- Repro: `pnpm mcporter generate-cli "node mock-stdio.mjs" --compile /tmp/inline-cli --runtime bun` using the inline stdio harness in `tests/cli-generate-cli.integration.test.ts`.
- Status: reproduced locally; pinned the SDK to `~1.21.2` until upstream ships a fix.

If you run into other recurring pain points, append them here so we can prioritize fixes.
