# Deploying the MCP Server to Azure App Service

How to deploy `SMIP_MCP/smip_mcp_server.py` to an Azure Web App so Azure
AI Foundry agents (or any MCP client) can use it.

## Architecture

```
MCP Client (AI Foundry / Claude Desktop / Cursor)
        │
        ▼  HTTPS
Azure App Service (reverse proxy, terminates TLS)
        │
        ▼  HTTP :8000
uvicorn  →  _RewriteHostMiddleware  →  mcp.streamable_http_app() | mcp.sse_app()
        │
        ▼  GraphQL + JWT
ThinkIQ / SMIP tenant
```

The MCP server is just a tool provider — there is no LLM in this
deployment. The model lives in the client (Foundry, etc.) and calls
tools over HTTPS.

## Prerequisites

- Azure subscription with an App Service plan (B1 or higher; F1 has no Always On).
- Python 3.12 runtime selected on the Web App, Linux OS.
- Azure App Service extension for Cursor / VS Code (for the easy deploy path)
  **or** Azure CLI (`az`) signed in (`az login`).
- SMIP authenticator credentials for your ThinkIQ tenant (`graphQlEndpoint`,
  `clientId`, `clientSecret`, `role`, `userName`).

## Files that matter for deployment

| File | Purpose |
|---|---|
| `SMIP_MCP/smip_mcp_server.py` | MCP server entry point. Mounts the tools, applies `_RewriteHostMiddleware`, runs uvicorn. |
| `SMIP_MCP/smip_tools.py` | `TOOL_REGISTRY` — single source of truth for every tool. |
| `SMIP_IO/smip_client.py` | SMIP transport. Reads `config.json` locally, falls back to env vars on Azure. |
| `SMIP_IO/smip_methods.py` | GraphQL query logic used by every tool. |
| `requirements.txt` | Python deps. Includes `mcp`, `uvicorn`, `starlette`. **No** gunicorn. |
| `startup.sh` | One-line Azure startup command (`exec python SMIP_MCP/smip_mcp_server.py`). |
| `.vscode/settings.json` | `appService.zipIgnorePattern` — what the Cursor extension excludes from the deploy zip. |

## Step 1 — Create the Web App (skip if it already exists)

Azure Portal → **App Services → Create**:

- **Runtime**: Python 3.12
- **OS**: Linux
- **Pricing**: B1 minimum (Always On requires it)

## Step 2 — Add Application Settings

Web App → **Settings → Environment variables → Application settings**.
Add each row, then click **Apply** at the bottom.

| Name | Value |
|---|---|
| `SMIP_GRAPHQL_ENDPOINT` | `https://<your-instance>.thinkiq.net/graphql` |
| `SMIP_CLIENT_ID` | (from your authenticator) |
| `SMIP_CLIENT_SECRET` | (from your authenticator) |
| `SMIP_ROLE` | (from your authenticator) |
| `SMIP_USERNAME` | (from your authenticator) |
| `MCP_TRANSPORT` | `sse` (recommended) or `streamable-http` |
| `SCM_DO_BUILD_DURING_DEPLOYMENT` | `true` |
| `WEBSITES_PORT` | `8000` |

Endpoint per transport:

- `sse`             → `https://<your-app>.azurewebsites.net/sse`
- `streamable-http` → `https://<your-app>.azurewebsites.net/mcp`

You can switch transports by changing `MCP_TRANSPORT` and saving. The app
restarts automatically — no redeploy needed.

> Do NOT add Azure OpenAI variables here. The MCP server doesn't use the
> LLM; the client does. Those vars are only for the Flask `/api/chat`
> surface, which is excluded from this deploy.

## Step 3 — Set the Startup Command

Web App → **Settings → Configuration → General settings**:

| Field | Value |
|---|---|
| Stack | Python |
| Major version | 3.12 |
| Startup Command | `python SMIP_MCP/smip_mcp_server.py` |
| Always On | On |

Save. The app restarts.

> Do NOT use `gunicorn`. Gunicorn's Starlette wrappers don't propagate
> the MCP SDK's ASGI lifespan, so the session manager never initializes.
> Run uvicorn directly via the Python entry point — that is what the
> startup command above does.

## Step 4 — Deploy

### Option A — Cursor / VS Code Azure extension (easy path)

1. Make sure `.venv/` is **outside** the workspace, **or** confirm
   `.vscode/settings.json` lists it under `appService.zipIgnorePattern`
   (it does in this repo). The extension does not respect `.gitignore`
   or `.deployignore` for App Service deploys — only `appService.zipIgnorePattern`.
2. **Ctrl+Shift+P → Azure App Service: Deploy to Web App…**
3. Folder: project root. Web App: your target. Confirm overwrite.
4. Watch **Output panel → Azure App Service**. The first lines should be:

    ```
    Creating zip package...
    Ignoring files from "appService.zipIgnorePattern"
    "__pycache__{,/**}"
    ".venv{,/**}"
    ...
    ```

   If you see that block with `.venv{,/**}` listed, exclusions are
   working and the deploy will finish in 1–2 minutes. If you don't see
   it, fall back to Option B.

### Option B — Azure CLI (deterministic, recommended for CI)

Build a clean zip yourself, then push it.

```powershell
$staging = "$env:TEMP\mcp-deploy"
Remove-Item -Recurse -Force $staging -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Path $staging | Out-Null

Copy-Item -Recurse SMIP_IO  "$staging\SMIP_IO"
Copy-Item -Recurse SMIP_MCP "$staging\SMIP_MCP"
Copy-Item requirements.txt  "$staging\"
Copy-Item startup.sh        "$staging\"

# Strip secrets and caches
Get-ChildItem -Recurse -Force -Path $staging `
  -Include __pycache__,*.pyc,config.json -ErrorAction SilentlyContinue `
  | Remove-Item -Recurse -Force -ErrorAction SilentlyContinue

$zip = "$env:TEMP\mcp-deploy.zip"
Remove-Item $zip -ErrorAction SilentlyContinue
Compress-Archive -Path "$staging\*" -DestinationPath $zip
"Zip size: {0:N2} MB" -f ((Get-Item $zip).Length / 1MB)   # expect < 1 MB

$rg = az webapp show --name <your-app> --query resourceGroup -o tsv
az webapp deploy --resource-group $rg --name <your-app> --src-path $zip --type zip
```

The CLI streams the Oryx + pip install output, so you see exactly where
it fails if it does.

## Step 5 — Verify

### 5a. Logs (live stream)

```powershell
az webapp log tail --name <your-app> --resource-group $rg
```

Look for:

```
INFO:     Started server process [...]
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000
```

### 5b. Public endpoint

For SSE:

```powershell
curl.exe -N https://<your-app>.azurewebsites.net/sse
```

Expect an SSE stream with `event: endpoint` followed by periodic pings.

For Streamable HTTP:

```powershell
curl.exe -X POST `
  -H "Accept: application/json, text/event-stream" `
  -H "Content-Type: application/json" `
  -d '{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"initialize\",\"params\":{\"protocolVersion\":\"2025-06-18\",\"capabilities\":{},\"clientInfo\":{\"name\":\"curl\",\"version\":\"1.0\"}}}' `
  https://<your-app>.azurewebsites.net/mcp
```

Expect a JSON-RPC `result` containing `serverInfo: {"name": "TAF Carbon Automation", ...}`.

### 5c. Real MCP round-trip from Python

`test_mcp_remote.py` in the repo root does a full handshake, lists the
tools, and calls one read-only tool plus one SMIP-touching tool. Edit
the `URL` constant at the top to match your app, then:

```powershell
.\.venv\Scripts\Activate.ps1
python test_mcp_remote.py
```

A clean run prints the server name, the 9 tool names, the glossary
markdown, and a JSON list of storage shelves — that's the proof that
auth + GraphQL + tool dispatch all work end-to-end on the deployed app.

## Step 6 — Connect to Azure AI Foundry

In your Foundry agent's MCP tool config:

| Field | Value |
|---|---|
| Server URL | `https://<your-app>.azurewebsites.net/sse` (or `/mcp`) |
| Transport | Match the URL (SSE or Streamable HTTP) |

Foundry auto-discovers all 9 tools and exposes them to the agent.

## Gotchas

### 1. Cursor / VS Code extension zips the entire workspace

The extension does **not** read `.gitignore` or `.deployignore`. It
reads `appService.zipIgnorePattern` in `.vscode/settings.json`. If you
have a local `.venv/` in the workspace and that pattern is missing,
the extension uploads ~100 MB of Windows wheels and Kudu times out at
the 10-minute mark.

**Fix**: this repo already has the right `.vscode/settings.json`. If a
fresh clone deploys cleanly but a working copy doesn't, check that
file is intact.

### 2. MCP SDK rejects Azure's Host header

The SDK validates `Host` and only accepts `localhost` by default
(transport-level DNS-rebinding protection). Azure's reverse proxy
forwards the public hostname.

**Fix**: `_RewriteHostMiddleware` in `smip_mcp_server.py` rewrites Host
to `localhost:<port>` before the request reaches the MCP app. Don't
remove that wrapper.

### 3. Gunicorn breaks the MCP SDK lifespan

Gunicorn + Starlette wrappers swallow the lifespan context, so the MCP
session manager never initializes — connections succeed but every
request returns 500 with a `session manager not initialized` error.

**Fix**: the startup command must be `python SMIP_MCP/smip_mcp_server.py`.
That runs uvicorn directly, which propagates lifespan correctly.

### 4. `config.json` is gitignored

`SMIP_IO/config.json` holds tenant secrets and is excluded from both
git and the deploy zip.

**Fix**: `SMIPClient` falls back to env vars (`SMIP_GRAPHQL_ENDPOINT`,
`SMIP_CLIENT_ID`, `SMIP_CLIENT_SECRET`, `SMIP_ROLE`, `SMIP_USERNAME`)
when the file is absent. Set them as App Settings (Step 2).

### 5. Streamable HTTP has known issues on App Service

GitHub issue [microsoft/mcp#1736](https://github.com/microsoft/mcp/issues/1736)
documents intermittent failures.

**Fix**: default `MCP_TRANSPORT=sse`. If you want Streamable HTTP, try
it — but if Foundry hangs or errors mid-call, flip back to SSE in App
Settings. No redeploy needed; the app restarts on save.

### 6. Single instance only

The SMIPClient caches a JWT in memory and the MCP SDK's session manager
is per-worker. Don't autoscale beyond one instance unless you also
enable ARR Affinity (sticky sessions).

## Adding a new tool (no deploy changes needed)

The fan-out pattern is three edits:

1. New method on `SMIPMethods` in `SMIP_IO/smip_methods.py`.
2. New entry in `TOOL_REGISTRY` in `SMIP_MCP/smip_tools.py`
   (`name`, `summary`, `description`, `parameters`, `ui`, `fn`).
3. New typed `@mcp.tool()` wrapper in `SMIP_MCP/smip_mcp_server.py`
   (3 lines — FastMCP reads the signature for the JSON schema).

Redeploy via Step 4. Foundry picks up the new tool on the next session
refresh; no client config change required.
