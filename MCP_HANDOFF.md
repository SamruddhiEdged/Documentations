# MCP Server Handoff Notes


---


## 2. Python environment setup

### Local development

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

### Python version

- Locally we developed on **Python 3.14** (whichever the system `python`
  resolves to is fine; nothing in the code requires 3.14 specifically).
- On Azure App Service we use **Python 3.12**. The `requirements.txt`
  has no version-specific deps, so 3.11 / 3.12 / 3.13 / 3.14 all work.
- Important: **never deploy the local `.venv/`**. It contains
  Windows-platform wheels (`cp314-cp314-win_amd64.whl`) that crash on
  Azure's Linux containers. Oryx builds a fresh venv on the server from
  `requirements.txt` because `SCM_DO_BUILD_DURING_DEPLOYMENT=true`.

### Azure runtime (Oryx-managed)

Azure App Service does this on every deploy:

1. Receives the zip uploaded by Cursor extension or `az webapp deploy`.
2. Sees `requirements.txt` at the zip root → identifies the app as Python.
3. Runs `pip install -r requirements.txt` into a managed virtualenv at
   `/tmp/<deployment-id>/antenv`.
4. Starts the app via the configured Startup Command (Step 3 of
   `DEPLOY_AZURE.md`).

You don't need to commit a venv, lock file, or anything Python-specific
beyond `requirements.txt`. If you add a dep, add it to that file.

---

## 3. Code changes — file by file

The diff between "fresh fork - just MCP" and "deployable MCP" is six files. Each
is small. Here's what each changed and why.

### 3a. `SMIP_IO/smip_client.py` — env-var fallback

**Before**: raised `FileNotFoundError` if `SMIP_IO/config.json` was
missing.

**After**: when `config.json` is absent, reads from environment
variables instead:

| Env var | Maps to |
|---|---|
| `SMIP_GRAPHQL_ENDPOINT` | `graphQlEndpoint` |
| `SMIP_CLIENT_ID` | `clientId` |
| `SMIP_CLIENT_SECRET` | `clientSecret` |
| `SMIP_ROLE` | `role` |
| `SMIP_USERNAME` | `userName` |

Local dev still uses `config.json` (which beats env vars when both are
present). On Azure we set the env vars as App Settings — no secrets in
the zip. Error messages tell you which source was tried so a
misconfiguration is obvious from the logs.

### 3b. `SMIP_MCP/smip_mcp_server.py` — middleware + uvicorn-direct entry

Three changes near the bottom of the file:

1. **Added `_RewriteHostMiddleware`**. ASGI middleware that rewrites the
   inbound `Host` header to `localhost:<PORT>` before the request reaches
   the MCP app. Keeps the SDK's transport-security check intact while
   letting Azure's reverse proxy do its job.
2. **Removed the module-level `app = mcp.sse_app()`** that was there for
   gunicorn. We don't expose an ASGI app at module scope anymore — the
   transport (and therefore which app object to build) is decided at
   runtime from `MCP_TRANSPORT`.
3. **Rewrote the entry point** to a transport switch:

    ```python
    transport = os.environ.get('MCP_TRANSPORT', 'stdio').lower()

    if transport == 'stdio':
        mcp.run(transport='stdio')
    elif transport in ('sse', 'streamable-http'):
        inner_app = mcp.sse_app() if transport == 'sse' else mcp.streamable_http_app()
        app = _RewriteHostMiddleware(inner_app, host=f"localhost:{port}")
        uvicorn.run(app, host='0.0.0.0', port=port, log_level='info')
    ```

   This is what Azure's startup command (`python SMIP_MCP/smip_mcp_server.py`)
   ends up running. uvicorn-direct (no gunicorn) is what makes the MCP
   SDK's lifespan + session manager work.

### 3c. `requirements.txt` — gunicorn out, starlette in

Removed `gunicorn` (we don't use it). Added `starlette` (used by the
new `_RewriteHostMiddleware` for the ASGI types). Final list:

```
flask
python-dotenv
openai
requests
mcp
uvicorn
starlette
openpyxl
```

### 3d. `startup.sh` — Azure startup command

One-line wrapper around the entry point. Azure runs whatever you put
in **Configuration → General settings → Startup Command**; we point
that at `python SMIP_MCP/smip_mcp_server.py` directly, so this file
exists mostly as documentation / fallback.

### 3e. `.deployignore` — `az` CLI exclude list

Used by `az webapp up` and some other Azure CLI flows. Excludes the
non-MCP folders, secrets, caches, and the `.venv/`.

### 3f. `.vscode/settings.json` — Cursor extension excludes

The single most important file for whoever deploys next. The Cursor /
VS Code Azure App Service extension reads
`appService.zipIgnorePattern` from this file to decide what NOT to
zip. **It does not read `.gitignore` or `.deployignore`.** Without this,
the extension uploads `.venv/`, `.git/`, `___SMIP_SAAS_SIDE___/`, etc.
— well over 100 MB — and Kudu times out.

If you ever delete this file or change its name, the deploy will
silently revert to "zip the entire workspace" behavior.

### 3g. (Optional) `test_mcp_remote.py`

A standalone end-to-end test script in the repo root. Edit the `URL`
constant at the top, then `python test_mcp_remote.py`. Performs the
MCP `initialize` handshake, lists every tool, and calls one tool that
hits SMIP. Useful as a smoke test after any deploy.

---

## 4. Transports — SSE vs Streamable HTTP

The MCP spec has multiple transports. The SDK supports three; this
server exposes all three.

| `MCP_TRANSPORT` value | Endpoint | Use case |
|---|---|---|
| `stdio` (default) | n/a | Local clients (Claude Desktop, Cursor, custom agents) launching the script as a subprocess. |
| `sse` | `/sse` | Server-Sent Events. Long-lived stream, simpler protocol, broader compatibility. **Recommended for Azure App Service.** |
| `streamable-http` | `/mcp` | Newer Streamable HTTP transport. Theoretically more efficient. |

### Was Streamable HTTP supported?

Yes — `MCP_TRANSPORT=streamable-http` works and the endpoint is
`/mcp`. The MCP Python SDK has a top-level `mcp.streamable_http_app()`
that returns the right ASGI app, and the entry point in
`smip_mcp_server.py` calls it when the env var is set to that value.

**Caveat**: Streamable HTTP has known intermittent issues on Azure App
Service — see [microsoft/mcp#1736](https://github.com/microsoft/mcp/issues/1736).
We deployed with it once successfully, but if Foundry hangs or errors
mid-call after a Streamable HTTP deploy, the recovery is one env var:

> Settings → Environment variables → set `MCP_TRANSPORT=sse` → Apply.

The app restarts in ~30s, the endpoint becomes `/sse`, you update the
client URL, and you're back. No code change, no redeploy.

### Switching transports without redeploying

This is the design payoff — transport is a runtime decision. To switch:

1. Change `MCP_TRANSPORT` in App Settings.
2. Click **Apply** (Azure restarts the app automatically).
3. Update your client to point at the new endpoint (`/sse` ↔ `/mcp`).

That's it.

---

## 5. Azure App Service configuration reference

Quick reference. Full step-by-step lives in `DEPLOY_AZURE.md`.

### 5a. Application Settings (env vars on the Web App)

| Name | Value | Required? | Why |
|---|---|---|---|
| `SMIP_GRAPHQL_ENDPOINT` | `https://<tenant>.thinkiq.net/graphql` | Yes | SMIPClient env-var fallback. |
| `SMIP_CLIENT_ID` | from authenticator | Yes | SMIPClient env-var fallback. |
| `SMIP_CLIENT_SECRET` | from authenticator | Yes | SMIPClient env-var fallback. |
| `SMIP_ROLE` | from authenticator | Yes | SMIPClient env-var fallback. |
| `SMIP_USERNAME` | from authenticator | Yes | SMIPClient env-var fallback. |
| `MCP_TRANSPORT` | `sse` or `streamable-http` | Yes | Picks endpoint shape. Default in code is `stdio` which is wrong for App Service. |
| `WEBSITES_PORT` | `8000` | Yes | Tells App Service which container port to route 443 → to. Must match what uvicorn binds (also `8000` via the `PORT` env var Azure sets, fallback 8000). |
| `SCM_DO_BUILD_DURING_DEPLOYMENT` | `true` | Yes | Makes Oryx run `pip install -r requirements.txt` on deploy. Without this you get `ModuleNotFoundError: No module named 'mcp'`. |
| `PORT` | (unset, optional) | No | Azure injects this automatically. Override only if you want a non-8000 port. |

### 5b. Startup Command

In **Configuration → General settings**:

```
python SMIP_MCP/smip_mcp_server.py
```

Reasons:

- It's the only thing that runs uvicorn directly (no gunicorn → MCP
  lifespan works).
- It reads `MCP_TRANSPORT` and `PORT` from the environment, so
  changing transport is just an App Setting flip.
- `startup.sh` would also work (it just `exec`s this same line). The
  Portal's "Startup Command" field is simpler than committing a script.

### 5c. Stack / Always On

- **Stack**: Python 3.12 (Linux).
- **Always On**: enabled. Required so the worker stays warm and SSE
  connections aren't reaped after idle.
- **Plan tier**: B1 minimum (Always On unavailable below B1). For
  production, P1V3 or higher is recommended for headroom on long-lived
  SSE connections.

### 5d. Single instance constraint

- Number of workers in the startup command: **1**. The SMIPClient
  caches a JWT in memory; multi-worker would each maintain their own.
- Number of instances: **1** unless ARR Affinity (sticky sessions) is
  enabled — SSE / Streamable HTTP sessions are stateful per worker.

---

## 6. What I'd do if I hit a brand new bug after handoff

In rough priority order:

1. **Check the App Service log stream first.**
   `az webapp log tail --name <app> --resource-group <rg>` shows the
   exact uvicorn / Python output. 95% of bugs reveal themselves here.
2. **Confirm App Settings are intact.** Easy to delete one by accident
   in the Portal. The `SMIP_*` set is the most common culprit.
3. **Reproduce locally first.** `MCP_TRANSPORT=sse python SMIP_MCP/smip_mcp_server.py`
   then `curl http://localhost:8000/sse`. If local fails, the deploy
   environment is fine and you have a code bug.
4. **Run `test_mcp_remote.py` against the deployed URL.** Tells you
   whether handshake / list_tools / call_tool work in isolation, before
   blaming Foundry.
5. **Last resort: rebuild the deploy zip yourself** and use
   `az webapp deploy --src-path ...` (Option B in `DEPLOY_AZURE.md`).
   Removes the Cursor extension from the equation if you suspect it.

---

## 7. Adding a new tool — the three-edit pattern

The whole point of the `TOOL_REGISTRY` is that adding a tool is one
add-only change in three files:

1. `SMIP_IO/smip_methods.py` — new method on `SMIPMethods` that does
   the GraphQL round-trip and returns plain Python (list of dicts).
2. `SMIP_MCP/smip_tools.py` — new entry in `TOOL_REGISTRY`:
   `name`, `summary`, `description`, `parameters`, `ui`, `fn` lambda.
3. `SMIP_MCP/smip_mcp_server.py` — new typed `@mcp.tool()` wrapper
   (3 lines) so FastMCP can build the JSON schema from the signature.

Then redeploy via the steps in `DEPLOY_AZURE.md`. Foundry picks up the
new tool on its next session refresh — no client config change needed.

For a worked example, look at any existing tool: `get_porosity_results`
is a typical "scope by node_id" read tool;
`get_storage_shelves_and_samples` shows the "name filter + boolean
flag" pattern.

---

## 8. Files at a glance

```
SMIP_IO/
  smip_client.py        # JWT auth + GraphQL POST. Reads config.json | env vars.
  smip_methods.py       # one method per business question.
  config.json           # gitignored; tenant credentials for local dev.

SMIP_MCP/
  smip_tools.py         # TOOL_REGISTRY (single source of truth).
  smip_mcp_server.py    # FastMCP server + _RewriteHostMiddleware + uvicorn entry.
  agent_prompt.py       # SYSTEM_INSTRUCTIONS shared with chat agent.

requirements.txt        # Python deps. NO gunicorn.
startup.sh              # Azure startup wrapper.
.vscode/settings.json   # Cursor extension zip excludes.  ← do not delete.
.deployignore           # az CLI zip excludes.
test_mcp_remote.py      # standalone end-to-end smoke test.

docs/
  DEPLOY_AZURE.md       # step-by-step deploy recipe.
  MCP_HANDOFF.md        # this file — context, decisions, gotchas.
  ARCHITECTURE.md       # the broader VibeCon-SMIP architecture.
  QUICKSTART.md         # local-dev quickstart (pre-existing).
  WORKFLOW.md           # how to use the template well (pre-existing).
```

