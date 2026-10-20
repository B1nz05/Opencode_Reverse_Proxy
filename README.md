# opencode-reverse-proxy

A local OpenAI-compatible HTTP gateway for Windows. Point any OpenAI SDK client
at it, and the request is routed through the **genuine OpenCode Desktop client**
to the free Zen models on your account.

Nothing is spoofed. The gateway does not hold credentials of its own — it reads
the local OpenCode service registration and calls the same background service the
desktop app uses, under your real login, billed to your own quota.

```
external client ──HTTP──▶ gateway (127.0.0.1:8787/v1)
                              │ HTTP Basic, read from service.json
                              ▼
                   OpenCode Desktop background service
                              │ real client identity + your login
                              ▼
                   opencode.ai free models
```

## What it is

This folder is an **OpenCode skill**. Copy it into
`%USERPROFILE%\.config\opencode\skills\opencode-reverse-proxy` and an OpenCode
agent can install, start, verify, troubleshoot, and remove the gateway for you.

It also works standalone: run the scripts directly, no agent required.

## Requirements

| Requirement | Notes |
|---|---|
| Windows | The scripts are PowerShell |
| Node.js 18+ | `node --version` |
| OpenCode Desktop | Installed, running, signed in, with free models visible in the app |

If the OpenCode model list has no free models, the account has no free tier and
the gateway cannot work around that.

## Quick start

```powershell
# 1. register the agents with OpenCode
powershell -ExecutionPolicy Bypass -File scripts\install.ps1

# 2. start the gateway (detached, returns immediately)
powershell -ExecutionPolicy Bypass -File scripts\start.ps1

# 3. prove the whole chain works end to end
powershell -ExecutionPolicy Bypass -File scripts\verify.ps1
```

Then point a client at it:

| Setting | Value |
|---|---|
| Base URL | `http://127.0.0.1:8787/v1` |
| API key | any non-empty value |
| Model | any id from `GET /v1/models` |

## Model routing

Append `@<agent>` to a model id to choose how much the model is allowed to do.

| Model spec | Capability |
|---|---|
| `big-pickle` | Text only, every dangerous tool denied (agent `orp-chat`) |
| `big-pickle@orp-writer` | Read and write files inside the sandbox only |
| `big-pickle@orp-full` | Unrestricted: whole disk, shell, network |

Query `/v1/models` for the current model ids — they change over time. The base
name shown here is just an example.

## Naming

| Thing | Value |
|---|---|
| Product / skill ID | `opencode-reverse-proxy` |
| Environment variables | `ORP_*` |
| Agents | `orp-chat`, `orp-writer`, `orp-full` |
| Writer sandbox | `%USERPROFILE%\Downloads\Opencode Reverse Proxy` |
| Default port | `8787` |
| Global config | `%USERPROFILE%\.config\opencode\opencode.json` |
| Runtime state | `%LOCALAPPDATA%\opencode-reverse-proxy\` |

## Commands

| Script | Purpose |
|---|---|
| `scripts\install.ps1` | Create the sandbox, merge the agents into the global config, verify registration. Options: `-WriterDir`, `-NoRestart` |
| `scripts\start.ps1` | Start the gateway detached and wait for health. Options: `-Port`, `-Key`, `-DefaultModel` |
| `scripts\stop.ps1` | Stop the gateway. Option: `-Port` |
| `scripts\verify.ps1` | Full end-to-end check. Options: `-InstallOnly`, `-SkipWriteTest`, `-Model`, `-Port` |
| `scripts\uninstall.ps1` | Stop and remove the agents. Option: `-IncludeLegacy` |

## Environment variables

| Variable | Default | Meaning |
|---|---|---|
| `ORP_PORT` | `8787` | Listen port |
| `ORP_HOST` | `127.0.0.1` | Listen address. Do not set `0.0.0.0` |
| `ORP_KEY` | none | When set, clients must send `Authorization: Bearer <key>` |
| `ORP_AGENT` | `orp-chat` | Agent used when the model has no `@` suffix |
| `ORP_AGENTS` | `orp-chat,orp-writer,orp-full` | Allowlist for the `@` suffix |
| `ORP_DEFAULT_MODEL` | first available | Model used when the caller omits one |
| `ORP_TIMEOUT_MS` | `300000` | Per-request ceiling |
| `ORP_KEEP_SESSIONS` | off | `1` keeps the OpenCode sessions instead of deleting them |
| `ORP_STRIP_TOOLS` | `1` via `start.ps1` | Accept `tools` requests but answer as text |
| `ORP_SERVICE_JSON` | `%USERPROFILE%\.local\state\opencode\service.json` | Service registry path |

## Limitations

- **No tool calling.** The OpenCode session API is agentic and cannot execute
  external OpenAI tool definitions. Requests with `tools`/`tool_choice` are
  answered as plain text and marked with `reverse_proxy.tools_ignored: true`
  (streaming: `X-ORP-Tools-Ignored: 1`). No fake `tool_calls` are ever returned.
  Clients that need real tool calls, such as Cline's tool panel, will not work.
- **`@orp-writer` needs absolute paths.** The gateway forwards only the
  conversation text, so the model does not know its working directory.
- **`glob` is inert on `orp-writer`.** In OpenCode V2 the `glob` permission
  matches the requested pattern, not a directory, so a path-shaped rule never
  matches. Documented, not fixed.
- **Only chat completions.** `/v1/responses`, `/v1/messages`, and
  `/v1/embeddings` return a clear 400.
- **Images are not forwarded.** They become `[binary image content omitted]`.
- **Quota errors pass through.** 429/403 are reported as-is.

## Security

The gateway binds `127.0.0.1` only, but by default it does **not** check an API
key, and `orp-full` has no permission restrictions. Any local process can
therefore read and write the whole disk, and run shell commands, as the logged-in
user, through `@orp-full`.

If that is not acceptable:

```powershell
powershell -ExecutionPolicy Bypass -File scripts\start.ps1 -Key "<long-random-string>"
```

Removing the `orp-full` entry from `agents.json` before installing is the other
half of the fix.

## Going deeper

- `SKILL.md` — instructions for an OpenCode agent using this folder as a skill
- `references/client-setup.md` — curl, Python, Node, and route support
- `references/permissions.md` — the exact agent rule model
- `references/troubleshooting.md` — symptom-by-symptom fixes

## Uninstall

```powershell
powershell -ExecutionPolicy Bypass -File scripts\uninstall.ps1
```

Then delete this folder, the sandbox folder, and
`%LOCALAPPDATA%\opencode-reverse-proxy`.
