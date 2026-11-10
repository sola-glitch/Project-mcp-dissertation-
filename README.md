# MCP Orchestration

A hub-and-spoke tool orchestration system. A central **hub** discovers a set of small tool servers (**spokes**), routes each tool call to the spoke that owns it, logs every call to an audit log, and saves session state so work can be resumed.

You can drive it four ways:

- the **web UI** (a browser control panel),
- the **terminal agent** (`agent.py`),
- an optional built-in **AI assistant** that uses the tools for you, with **your own model and API key**,
- any **MCP client** (Claude Desktop, Cursor, VS Code, ...) through the included **MCP server**.

And you can run all of it inside a **locked-down Docker container**.

```
 Browser (web UI)     Terminal (agent.py)     MCP clients (stdio)
        \                   |                       /
     hub_api.py  --->  hub.py  <---  mcp_server.py
     (login token)       |
                         +--> spokes (tools/*_server.py, ports 8001-8009, shared secret)
                         +--> ai_agent.py (optional, your own key, streaming, saved chats)
                         +--> encrypted storage: credentials, sessions, chats (mcp_data.db)
```

---

## What it does

This is a control panel that lets you run tasks on your computer from one place instead of using a separate tool for each job.

- There are nine small helper programs (the **spokes**). Each does one job: run shell commands, read and write files, run Python code, query a database, call websites, talk to GitHub, and so on.
- One central program (the **hub**) knows about all of them. You ask for a tool, the hub sends the request to the right spoke and returns the result.
- Every call is written to an audit log, and sessions are saved so you can resume later.

**Example:** in the web UI, pick `run_python`, enter `print(1+1)` and press Run. The hub passes it to the code-execution spoke and shows `2`.

**About AI:** the hub works fine with no AI at all, because you pick the tool and give it the input. If you want an assistant to choose and run tools for you, you can switch on the optional AI assistant with **your own provider, model and API key** (see section 6). Nothing is sent to any AI service unless you set that up.

## You choose your own model and API key (optional)

This project ships with **no API keys, tokens, accounts or default model**. Anything that talks to an outside service uses *your own* credentials:

- **AI assistant:** your own key for the provider you choose (Anthropic, OpenAI, or any OpenAI-compatible service), plus the model you want. Run `python check_ai.py` to confirm your key and model work before you rely on them.
- **GitHub spoke:** your own GitHub personal access token as `GITHUB_TOKEN`.

Keep keys in `.env` (or type them into the AI panel, which keeps them in memory only). Never put them in code, commit them, or share them. You are responsible for the usage and cost on your own keys.

## Secrets and logins (all created automatically)

| Secret | What it protects | Where it lives |
|---|---|---|
| **Hub access token** | The web UI and API. Every `/api/*` request needs it. | `HUB_API_TOKEN` in `.env`, or the `.hub_token` file |
| **Spoke shared secret** | The spokes. They refuse every call that doesn't carry it, so other programs on your computer can't use the shell or file spokes directly. Only the hub knows it. | `SPOKE_SECRET` in `.env`, or the `.spoke_secret` file |
| **Vault key** | Encrypts saved credentials, sessions and AI chats on disk (Fernet: AES with an integrity check). | `VAULT_KEY` in `.env`, or the `.vault_key` file |

All three files are created with owner-only permissions and are in `.gitignore`. `run_all.py` opens your browser already signed in; if you open the page yourself it asks you to paste the hub token once.

**Back up your vault key.** If you lose it, saved credentials, sessions and chats can no longer be read (the app will say so clearly; delete `mcp_data.db` to start fresh). Putting the key in `VAULT_KEY` instead of the file keeps it out of the data folder, which protects you better if someone gets a copy of `mcp_data.db`.

---

## 1. What you need

| Requirement | Details |
|---|---|
| **Python** | 3.10 or newer recommended (developed and tested on 3.12). 3.9 should work. |
| **pip** | Comes with Python. |
| **Free ports** | 8000 (web UI) and 8001 to 8009 (spokes) on `127.0.0.1`. |
| **A web browser** | Only for the web UI. |
| **Internet** | Only to install packages, and for the `rest_api`, `github` and AI features. |

Python packages are installed from `requirements.txt`: fastapi, uvicorn, httpx, sqlalchemy, python-dotenv, requests, PyGithub, cryptography, pytest, pytest-asyncio.

**Optional extras (only if you use that feature):**

- **An AI provider account and API key** for the AI assistant. Any provider that offers an Anthropic-style or OpenAI-style API works. Local servers such as Ollama need no key.
- **Docker** to run everything in a container (section 3), or to run each code snippet in its own throwaway container (section 8).
- **A GitHub token** for the `github` spoke (github.com > Settings > Developer settings > Personal access tokens).
- **Cursor editor with its `cursor` command on PATH** for the `cursor` spoke.
- **VS Code with its `code` command on PATH** for `connectors.open_in_vscode`.
- **Node.js** only to run the browser test (section 10).

---

## 2. Install

```bash
cd MCP_ORCHESTRATION
python -m venv venv

# activate it
source venv/bin/activate          # macOS / Linux
venv\Scripts\activate             # Windows

pip install -r requirements.txt
cp .env.example .env              # Windows: copy .env.example .env
```

Open `.env` and fill in only what you need (GitHub token, AI provider, key and model). Everything is optional.

On Termux, if `pip install cryptography` fails to build, run `pkg install python-cryptography` first.

---

## 3. Run it

### Option A: directly on your computer

```bash
python run_all.py
```

This starts all nine spokes, waits until each one is healthy, starts the hub API, and opens **http://127.0.0.1:8000** in your browser already signed in. Press **Ctrl+C** to stop everything.

Other ways to start it:

```bash
python run_all.py --spokes-only   # just the spokes (for agent.py or mcp_server.py)
python agent.py                   # terminal agent (run in a second terminal)
```

### Option B: in a locked-down Docker container (recommended when using the AI assistant)

```bash
cp .env.example .env              # fill in only what you need
mkdir -p workspace                # the only folder the file tools can see
docker compose up --build
```

Open **http://127.0.0.1:8000** and paste the access token:

```bash
docker compose exec hub cat /data/.hub_token
```

(or set `HUB_API_TOKEN` in `.env` and use that). What the container gives you:

- everything runs as a non-root user with **all Linux capabilities dropped**, `no-new-privileges`, a **read-only filesystem**, and memory and process limits
- the file tools can only touch `./workspace`, not your home folder
- the web port is published on `127.0.0.1` only, so nothing on your network can reach it
- the database, secrets and audit log live in a Docker volume and survive restarts
- the toolmaker spoke cannot write new spoke files (the container's code is read-only); create spokes on your computer, then rebuild
- code snippets run in the container's process mode. Docker mode (section 8) needs Docker on your computer, not inside the container

To use the container from an MCP client, point it at `docker compose exec`:

```json
{ "mcpServers": { "orchestration": {
    "command": "docker",
    "args": ["compose", "-f", "/full/path/to/docker-compose.yml", "exec", "-T", "hub", "python", "mcp_server.py"] } } }
```

---

## 4. Using the web UI

- **Spoke Wheel:** the hub with every spoke around it. Teal means reachable, red means not. The line to a spoke flashes when you call one of its tools.
- **Tool Catalog:** every discovered tool, grouped by spoke and searchable. Click a tool to load it into the console.
- **AI Assistant:** pick your provider, paste your key, choose a model, then chat. Replies stream in as they are written, and past chats are listed so you can reopen or delete them (see section 6).
- **Console:** enter a spoke, a tool and JSON params, then press Run. Example: spoke `shell`, tool `run_command`, params `{"command": "echo hello"}`.
- **Audit Log:** live view of `audit.log`. Calls made by the AI are tagged `(ai)`.
- **Refresh:** re-runs discovery after you start or stop spokes.

## 5. Using the terminal agent

Start the spokes (`python run_all.py --spokes-only`), then in another terminal run `python agent.py`.

| Command | What it does |
|---|---|
| `tools` | List every discovered tool |
| `call <spoke> <tool> <json>` | Call any tool directly, e.g. `call code_exec run_python {"code":"print(1+1)"}` |
| `ls [path]` / `cd <path>` | Browse files (filesystem spoke) |
| `run <command>` | Run a shell command (shell spoke) |
| `ask <message>` | Talk to the AI assistant (needs your provider, key and model in `.env`) |
| `sessions` / `resume <id>` / `new` | Manage saved sessions |
| `maketool <name> <port> <purpose>` | Scaffold a new spoke |
| `exit` | Quit |

---

## 6. The AI assistant (bring your own model and key)

The assistant reads your request, decides which of the hub's tools to use, runs them, and reports back. It works with any of these:

| Provider setting | Works with | Key needed? |
|---|---|---|
| `anthropic` | Anthropic's API | Yes |
| `openai` | OpenAI's API | Yes |
| `openai` + a **Base URL** | Any OpenAI-compatible service: Google Gemini, Groq, OpenRouter, Together, local Ollama or LM Studio, and others | Usually yes; local servers usually no |

Typical Base URLs (check your provider's documentation for the exact one):

- Ollama (local): `http://localhost:11434/v1`
- OpenRouter: `https://openrouter.ai/api/v1`
- Groq: `https://api.groq.com/openai/v1`
- Gemini: `https://generativelanguage.googleapis.com/v1beta/openai`

**Set it up in the web UI:** open the AI Assistant panel, choose the provider, paste your key, press **Load models** to list the models your key can use, choose one, and press **Save**. The key stays in the app's memory only. It is never written to disk and never sent back to the browser.

**Or set it in `.env`** (needed for the terminal agent): `AI_PROVIDER`, `AI_API_KEY`, `AI_MODEL`, and optionally `AI_BASE_URL`.

**Check that it works:**

```bash
python check_ai.py
```

It tests your settings, a plain reply, a streamed reply, and a tool call (the model must use a harmless built-in add-two-numbers tool correctly), and tells you exactly what to fix if something fails. It never prints your key and uses a few cents at most. Models that don't support tool use will fail the last step: pick another model.

**Streaming and saved chats.** Replies appear word by word. Every chat is saved, encrypted, in `mcp_data.db`, so it survives a restart: reopen it from the *Previous chats* list in the UI, even if it was waiting for your approval when you closed it. The 50 most recent chats are kept. Delete one with *Delete chat*.

**Approvals.** The assistant cannot run these without you clicking **Allow** (or answering `y` in the terminal) for that exact call:

- running commands or code (`run_command`, `run_python`)
- creating, changing or deleting files (`create_file`, `update_file`, `delete_file`)
- reading file contents or database rows (`read_file`, `query`), because files and the credentials table can hold secrets
- changing data or reaching outside your machine (`execute`, `http_request`, `create_issue`, `create_spoke`)

Listing and searching tools run without asking. You can change the lists in `hub.py` (`DANGEROUS_TOOLS`, `READS_DATA_TOOLS`). Everything the assistant runs is recorded in `audit.log`.

**What gets sent to your provider:** your messages, the tool descriptions, and the results of any tool the assistant runs. Treat that like any other data you send to that provider.

---

## 7. Use it from MCP clients (Claude Desktop, Cursor, VS Code, ...)

`mcp_server.py` exposes every hub tool through the Model Context Protocol over stdio. It implements the MCP **tools** feature (initialize, ping, tools/list, tools/call) and was checked against the official MCP Python SDK client. It does not implement MCP resources, prompts, or the HTTP transport.

1. Start the spokes: `python run_all.py --spokes-only` (or use the Docker option above)
2. Add the server to your MCP client's config, using the full path to your copy:

```json
{
  "mcpServers": {
    "orchestration": {
      "command": "python",
      "args": ["/full/path/to/MCP_ORCHESTRATION/mcp_server.py"]
    }
  }
}
```

Use your venv's Python as the `command` if you installed the packages in a venv. Add `"--safe"` to `args` (or set `MCP_SAFE_MODE=1`) to hide every tool that runs commands or code, changes files or data, or makes web requests. Your MCP client has its own approval prompts. Calls from MCP clients are tagged `mcp` in `audit.log`.

---

## 8. The spokes

| Spoke | Port | What it does |
|---|---|---|
| `rest_api` | 8001 | Make outbound HTTP requests |
| `database` | 8002 | Query / write the local SQLite database (`mcp_data.db`) |
| `code_exec` | 8003 | Run a Python snippet in a hardened subprocess, or in a Docker container |
| `github` | 8004 | List repos, list and create issues (needs `GITHUB_TOKEN`) |
| `cursor` | 8005 | Open files, folders and diffs in Cursor |
| `toolmaker` | 8006 | Scaffold a new spoke file when no tool exists |
| `filesystem` | 8007 | Read-only navigate, list, search, info |
| `crud_files` | 8008 | Create, read, update, delete files |
| `shell` | 8009 | Run shell commands (30 s timeout) |

File spokes are restricted to your home folder by default. Set `HUB_ALLOWED_ROOTS` in `.env` (folders separated by `:`, or `;` on Windows) to narrow or change that.

### The code runner: two modes

**`process` (default): a hardened subprocess.** Each snippet runs in its own empty temp folder with `python -I`, with a clean environment (your API keys, tokens and secrets are **not** passed to it), CPU, memory and file-size limits (Linux and macOS), a 10-second timeout that kills the whole process group, and output cut to the last 5000 characters. It can still read files you can read and use the network, so this stops accidents and casual abuse. **It is not a security boundary.**

**`docker`: real isolation.** Set `CODE_EXEC_SANDBOX=docker` in `.env`, install Docker, and run `docker pull python:3.12-slim` once. Each snippet then runs in a throwaway container with **no network**, a read-only filesystem, no capabilities, a non-root user, 256 MB of memory, 1 CPU and 64 processes. If Docker is selected but missing, the call fails; it never quietly falls back to the weaker mode.

### Adding your own spoke

1. Use the console/agent to call `toolmaker` > `create_spoke` (or `maketool` in the agent).
2. Fill in the TODOs in the new `tools/<name>_server.py`.
3. Add it to `SPOKE_REGISTRY` in `hub.py`.
4. Restart `run_all.py`. It starts any spoke listed in the registry automatically.

Each tool's JSON schema (used by the AI assistant and the MCP server) is built automatically from the function's type hints, so add type hints to your tool functions. New spokes get the shared-secret check automatically.

---

## 9. Settings reference

All optional, set in `.env` (see `.env.example` for comments):

| Setting | Default | Purpose |
|---|---|---|
| `AI_PROVIDER`, `AI_API_KEY`, `AI_MODEL`, `AI_BASE_URL` | none | AI assistant |
| `GITHUB_TOKEN` | none | GitHub spoke |
| `HUB_API_TOKEN` | generated | Web UI / API access token |
| `SPOKE_SECRET` | generated | Hub-to-spoke shared secret |
| `VAULT_KEY` | generated | Encryption key for credentials, sessions, chats |
| `CODE_EXEC_SANDBOX`, `CODE_EXEC_DOCKER_IMAGE` | `process` | Code runner mode and image |
| `HUB_DATA_DIR` | project folder | Where the database, audit log and secret files live |
| `HUB_ALLOWED_ROOTS` | your home folder | Folders the file tools may touch |
| `MCP_SAFE_MODE` | off | Hide risky tools from MCP clients |

---

## 10. Running tests

```bash
python -m pytest
```

About 180 tests, no servers or API keys needed. They use temporary files, never your real data or `.env`. They cover the hub (discovery, routing, audit log, sessions), encryption, the login token and spoke secret, the web API, the AI assistant (against fake providers, including streaming, the approval rules and saved chats), the MCP server (including a real stdio round trip), the code runner (real limits, timeouts and process cleanup), and every spoke: database, GitHub, REST, Cursor, files, shell, toolmaker.

**Browser test (optional).** One test drives the real web UI in a simulated browser against the running system. It is skipped unless Node.js is installed and you ran this once:

```bash
cd tests/ui && npm install
```

It then starts everything and a fake AI provider, so ports 8000 to 8009 and 9911 must be free (about 30 seconds).

## 11. Running things manually (optional)

`run_all.py` is a shortcut. You can also start any spoke yourself from the project folder:

```bash
uvicorn tools.shell_server:app --port 8009
uvicorn hub_api:app --port 8000
```

---

## 12. Troubleshooting

| Problem | Fix |
|---|---|
| `ModuleNotFoundError` | Activate the venv and run `pip install -r requirements.txt`. |
| A spoke shows `FAILED to start` | Its port is probably in use. Close the other program or change the port in `hub.py` and the spoke file. |
| A spoke is red in the UI | Start it, then press **Refresh**. |
| "spoke rejected the shared secret" | An old spoke process is still running with a different secret. Stop everything and start again. |
| The page asks for a token | Paste the contents of `.hub_token` (or your `HUB_API_TOKEN`). Starting with `run_all.py` signs you in automatically. |
| `python check_ai.py` fails | Read the line marked FAIL: it says whether the key, model, streaming or tool use is the problem. |
| AI: "check your API key" | The provider rejected the key. Check it, and that the provider setting matches it. |
| AI: "check the model name and base URL" | Press **Load models** and pick one from the list, or fix the Base URL. |
| AI: "Could not reach the provider" | Check your internet connection or, for local models, that the server is running. |
| "Could not decrypt stored data" | The vault key changed or was lost. Restore your `VAULT_KEY` / `.vault_key`, or delete `mcp_data.db` to start fresh. |
| `github` returns "No GitHub token" | Set `GITHUB_TOKEN` in `.env` and restart. |
| `cursor` returns "CLI not found" | Install Cursor and enable its shell command, or ignore that spoke. |
| `PermissionError ... outside allowed roots` | The path is outside the allowed folders. Change `HUB_ALLOWED_ROOTS`, or in Docker put the file in `./workspace`. |
| Code runner: "docker command was not found" | You set `CODE_EXEC_SANDBOX=docker` but Docker isn't installed. Install it, or unset the setting. |
| Code runner times out in Docker mode | Run `docker pull python:3.12-slim` once first. |
| MCP client shows no tools | Start the spokes first (`python run_all.py --spokes-only`) and use the full path in the config. |
| Install fails on Termux or an unusual platform | Some packages (pydantic-core, cryptography) may need a Rust toolchain or a system package. See section 2. |

---

## 13. Security: please read

**What protects you now**

- The hub API needs the access token; the spokes need the shared secret, so nothing else on your computer can drive them.
- Credentials, sessions and AI chats are encrypted at rest. Secret files are owner-only.
- The AI assistant needs your approval for anything that runs, changes, reads or sends data, and every call it makes is audited.
- The code runner gets a clean environment with resource limits, or a no-network container in Docker mode.
- The Docker setup runs the whole system as a non-root, capability-free, read-only container that only sees one folder.

**What still needs care**

- The **shell** and **crud_files** spokes can run commands and change files as *your user* (inside the container, as the container's user). Servers listen on `127.0.0.1` only. **Never expose these ports to the internet or a shared network.**
- The AI assistant can be tricked by text it reads (for example a malicious file or web page telling it what to do). That is why risky tools need your approval. **Read each approval request before pressing Allow.**
- **Process-mode code execution is not a security boundary**: it can still read your files and use the network. Use Docker mode or the container for untrusted code.
- Encryption protects against someone who gets only `mcp_data.db`. It does not protect against someone who gets both the database and the key file, or who can run code as you. Use `VAULT_KEY` to keep the key elsewhere.
- `audit.log` is plain text and records tool parameters (for example shell commands). Don't put secrets in commands.
- The `http_request` tool can reach any address your computer can, including your local network. It needs approval from the AI assistant, but not from you when you call it yourself.
- Never commit `.env`, `.hub_token`, `.spoke_secret` or `.vault_key`. All are in `.gitignore`.

## 14. Project layout

See [PROJECT_STRUCTURE.md](PROJECT_STRUCTURE.md) for a file-by-file description and [USAGE_GUIDE.md](USAGE_GUIDE.md) for the original usage notes.
