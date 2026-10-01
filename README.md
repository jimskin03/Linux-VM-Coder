# Linux VM Coder

A self-hosted MCP server that gives an MCP-compatible coding agent access to tools on your Linux VM: workspace files, shell commands, Git, search, patches, project context, and process management. It can also proxy the optional Google Colab MCP server.

The core tools run locally on the VM without requiring separate desktop companion applications. Treat this server as privileged: anyone who can call its tools may be able to read or change files and run commands as the service account.

## Features

- Filesystem operations, search, patches, and project-context loading
- Shell, Git, Node.js REPL, background processes, and checkpoints
- Streamable HTTP MCP endpoint for compatible clients
- Local admin UI for status and MCP upstream configuration
- Optional Google Colab MCP integration, enabled by default

## Requirements

- Linux
- Node.js 18 or newer
- Git (for Git tools)
- `uv` / `uvx` (for the optional Colab MCP server)

## Quick start

```bash
git clone https://github.com/jimskin03/Linux-VM-Coder.git
cd Linux-VM-Coder
cp .env.example .env
corepack enable
pnpm install --frozen-lockfile
```

Edit `.env` and set an absolute `WORKSPACE_PATH` to the project directory the agent should work in. Generate a strong MCP token and put it in `MCP_TOKEN`:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('base64url'))"
```

Build and start the server:

```bash
pnpm run build
pnpm start
```

By default the MCP server listens only on `127.0.0.1:3000`. Check that it is running with:

```bash
curl http://127.0.0.1:3000/health
```

When `MCP_TOKEN` is set, configure the client with the MCP endpoint `http://<host>:3000/mcp/<MCP_TOKEN>` (or the equivalent HTTPS URL if using a secure tunnel). The token is part of the URL: keep it private. An empty token disables MCP authentication and is not safe for a reachable network endpoint.

## Google Colab MCP

The default upstream configuration enables only `colab-mcp`. Install Astral `uv` on the VM so the server can launch `uvx`:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Open a new shell or add the installer’s binary directory to `PATH`, then verify both commands:

```bash
uv --version
uvx --version
```

The MCP process must inherit a `PATH` that contains `uvx`. The server configuration is in `profiles/mcp-upstream.json`; it runs the upstream package from `https://github.com/googlecolab/colab-mcp` through `uvx` and exposes its tools. Restart Linux VM Coder after changing the file.

Colab MCP bridges an agent to a Colab session in a browser; it is not a replacement for a Colab account/session. The upstream project also requires an MCP client that supports `notifications/tools/list_changed`. A headless VM without access to a compatible client/browser session may not be able to use Colab features even though the core Linux VM Coder tools still work.

## Connect from another machine

Keep the service bound to loopback. For a client on your workstation, an SSH local forward is a private option:

```bash
ssh -N -L 3000:127.0.0.1:3000 <user>@<vm-host>
```

Then connect the local MCP client to `http://127.0.0.1:3000/mcp/<MCP_TOKEN>`.

If a cloud-hosted client must reach the VM, use a secure, controlled HTTPS tunnel that forwards only to port `3000`, and retain `MCP_TOKEN` authentication. Do not expose the admin UI port (`3001`) through a tunnel or public firewall rule.

## Configuration

Important `.env` settings:

| Variable | Default | Purpose |
| --- | --- | --- |
| `HOST` | `127.0.0.1` | MCP bind address; keep loopback unless you have a deliberate network-security design. |
| `PORT` | `3000` | MCP HTTP port. |
| `MCP_TOKEN` | empty | Secret required in the MCP URL path. Set this before exposing the endpoint. |
| `WORKSPACE_PATH` | current directory | Default working directory and source for project instructions; it is **not** a filesystem sandbox or access boundary. |
| `ADMIN_PORT` | `3001` | Local-only admin UI port. Do not tunnel or publicly expose it. |
| `ADMIN_TOKEN` | empty | Additional admin UI bearer token. Set one if you use the admin UI. |
| `MCP_UPSTREAM_CONFIG` | `profiles/mcp-upstream.json` | JSON configuration for optional upstream MCP servers. |
| `SHELL_TIMEOUT` | `120` | Maximum run time for a shell command, in seconds. |
| `CHATGPT_TOOL_PROFILE` | `slim` | Selects the exposed built-in tool set (`slim` or `full`). |

The admin UI is available at `http://127.0.0.1:3001/ui` by default. It can inspect upstreams and write settings, so keep it on loopback and never forward its port to an untrusted network.

## Included tools

Tool availability depends on the selected tool profile. Built-in groups include:

- **Context:** `agent_status`, `project_context`
- **Files:** read/write/edit, directory listing, glob/grep search, patching, and checkpoints
- **Shell and processes:** commands, persistent shell state, background process management
- **Git:** status, diff, add/commit, branch, pull/push, and related operations
- **Upstream MCP:** optional tools proxied from configured servers (only Colab MCP is enabled in the default config)

## Development and tests

```bash
pnpm run build
pnpm test
pnpm run test:all
```

For integration tests that start local MCP processes, see the individual scripts in `scripts/` and ensure their documented test prerequisites are installed.

## Security

This server is a privileged remote-control interface for the VM. Before connecting a client:

1. Set a long, random `MCP_TOKEN` and do not share the full endpoint URL.
2. Keep `HOST=127.0.0.1`; use a private SSH forward or a trusted tunnel when remote access is needed.
3. Forward only the MCP port (`3000`). Never expose the admin UI port (`3001`).
4. Use a dedicated, least-privilege Linux account and set `WORKSPACE_PATH` to the intended project. Absolute paths and shell commands can still access anything that account can access; `WORKSPACE_PATH` only sets the default directory.
5. Keep `.env` and all tunnel credentials private; they must not be committed.
6. Stop the server or tunnel when remote access is no longer needed.

## License

MIT. See [LICENSE](LICENSE).
