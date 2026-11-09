# Install Sextant with an AI agent

This page is written for an AI coding agent (Claude Code, Codex, Cursor, Aider, or any
agent with a shell) acting on behalf of a human. If you are the agent: follow the steps in
order, run the verify command after each one, and stop to ask the human at every point
marked **ASK**. Do not skip verifies. Do not install system packages or change files outside
the install directory without asking. The whole run takes 5 to 10 minutes.

If you are the human: paste this into your agent:

> Read https://usesextant.com/agent-install.md and follow it to install Sextant on this
> machine. Ask me before anything that needs sudo, admin rights, or touches files outside
> the project folder.

The canonical copy of this file lives in the repository at `docs/AGENT-INSTALL.md`; the
copy at usesextant.com is republished from it on every release.

---

## 0. Detect the environment

Run and record:

```
python --version || python3 --version || py -3 --version
git --version
```

- Sextant needs **Python 3.11 or newer** and **git**. If Python is older or missing, **ASK**
  the human how they want it installed (python.org, `uv python install 3.12`, the OS package
  manager). Do not install it yourself.
- Note the OS and shell: Windows PowerShell, Windows Git Bash, macOS, Linux, or WSL. The
  venv paths below differ between Windows (`.venv\Scripts\`) and everything else
  (`.venv/bin/`). Git Bash on Windows uses the Windows layout with forward slashes:
  `.venv/Scripts/python.exe`.
- **ASK** where to install if the human has not said. Default: a `sextant` folder inside
  their projects or home directory. Never install inside another project's repository.

## 1. Get the code

```
git clone https://github.com/protechtus/sextant.git
cd sextant
```

Verify: `git log --oneline -1` prints a commit and `pyproject.toml` exists.

(When the package is on PyPI, `pipx install sextant-cp` or `uv tool install sextant-cp` is
the short path and skips steps 1 to 3. Until then, install from the clone. Never run
`pip install sextant`: that is an unrelated package.)

## 2. Create the virtual environment

Windows (PowerShell or Git Bash):

```
py -3 -m venv .venv
.venv\Scripts\python.exe -m pip install -U pip setuptools
```

macOS / Linux / WSL:

```
python3 -m venv .venv
.venv/bin/python -m pip install -U pip setuptools
```

Verify: `.venv/bin/python --version` (or `.venv\Scripts\python.exe --version`) prints 3.11+.

Known failure: on Debian/Ubuntu, `venv` fails with an `ensurepip` error when the
`python3-venv` package is missing. **ASK** before running
`sudo apt-get install -y python3-venv`. If the human cannot use sudo, use
`python3 -m venv --without-pip .venv` and bootstrap pip with `get-pip.py`, or `uv venv`.

## 3. Install

Use the venv's Python explicitly (do not rely on `activate`; it is shell-specific):

```
<venv-python> -m pip install -e ".[dev,telegram,summarizer]"
```

Verify: `<venv-python> -m pip show sextant-cp` prints the version, and
`<venv-python> -m atlas_plus --version` prints it too.

Known failure: a dependency resolver conflict. Retry with the pinned set:
`<venv-python> -m pip install -e ".[dev,telegram,summarizer]" -c constraints.txt`.

## 4. Initialise the memory corpus

```
<venv-python> -m atlas_plus.init_cmd
```

This creates `~/.memory` (the markdown corpus the human owns) and `~/.atlas` (state).
If the human wants them elsewhere, set `ATLAS_MEMORY_ROOT` and `ATLAS_STATE_DIR` **before**
this step, and keep setting them for every later command. `ATLAS_WORKSPACE_ROOT` re-roots
both at once.

Verify: `<venv-python> -m atlas_plus.doctor.cli` runs and shows the corpus path. On a
brand-new corpus the memory-quality rows can show red; that is expected for an empty corpus
and is not an error.

## 5. Connect the MCP server to the human's AI client

The server runs over stdio and needs no token in this mode. Register it with the venv's
Python so it works regardless of PATH.

**Claude Code** (run from inside the clone):

Windows:
```
claude mcp add -s user sextant -- "$PWD\.venv\Scripts\python.exe" -m atlas_plus.mcp.memory_server.server --stdio
```
macOS / Linux:
```
claude mcp add -s user sextant -- "$PWD/.venv/bin/python" -m atlas_plus.mcp.memory_server.server --stdio
```
Git Bash on Windows: use `"$(pwd -W)/.venv/Scripts/python.exe"` as the command.

If the human set `ATLAS_MEMORY_ROOT`, pass it **after the server name**:
`claude mcp add -s user sextant -e ATLAS_MEMORY_ROOT=/their/path -- <command>`. Putting
`-e` before the name fails on current Claude Code.

Verify: `claude mcp list` shows `sextant ... Connected`.

**Claude Desktop**: merge `deploy/claude_desktop_config.snippet.json` into the client's
config file (`%APPDATA%\Claude\claude_desktop_config.json` on Windows,
`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS), replacing the
placeholder path with the venv Python. **ASK** before editing that file; show the diff.

**Cursor and other MCP clients**: same command and arguments as above, in the client's MCP
settings. The server name `sextant` must be unique per machine; a second install needs a
different name.

**Session-start priming for Claude Code** (optional, recommended): the snippets
`deploy/session_start_hook.json` and `deploy/stop_hook.json` add hooks to
`~/.claude/settings.json`. **ASK** before editing settings; if a `hooks` key exists, add
the entries rather than replacing the key.

## 6. Prove it works (the round-trip)

Through the connected client, or by driving the server directly, do all three:

1. Call `memory_health`. Expect `ok` and the corpus path.
2. Call `memory_remember_tool` with scope `project:demo` and the text
   "Sextant installed by an agent on <date>." Expect a path under `projects/`.
3. Call `memory_search_tool` with query "installed by an agent". Expect the fact back with
   a confidence band.

Scope grammar for step 2: `project:<name>`, `reference:<topic>`, `preference:<topic>`. A bare
`reference` is rejected.

Without an MCP client at hand, the same check from the shell:

```
<venv-python> -m atlas_plus.mcp.memory_server.server --stdio
```
then send JSON-RPC `initialize`, `tools/list` (expect 30 tools), and the three calls above.
Or use the CLI: `<venv-python> -m atlas_plus search "installed by an agent"`.

## 7. Report to the human

Tell them, in this order: where it is installed, where the corpus lives, which client is
connected, what the round-trip returned, and anything you had to ask or work around. Point
them at `docs/GETTING-STARTED.md` for daily use and `docs/USAGE.md` for the tool reference.

## Troubleshooting (seen in real installs)

| Symptom | Cause | Fix |
| :- | :- | :- |
| `Invalid environment variable format: sextant` from `claude mcp add` | `-e` placed before the server name | put the name first, then `-e KEY=VAL`, then `--` |
| `Scope 'reference' requires a topic` | bare scope | use `reference:<topic>` |
| Session start says "not in a known project directory" | the hook primes a project note only when the working directory is a project under `ATLAS_PROJECTS_ROOT` (default `~/projects`) | set `ATLAS_PROJECTS_ROOT`, or use `atlas-prime <task>` for task context |
| HTTP server refuses to start | `sextant-mcp` over HTTP needs `ATLAS_MCP_TOKEN`; stdio mode does not | use stdio for clients; set the token only for HTTP |
| Corpus "vanished" | `ATLAS_WORKSPACE_ROOT` was set after init | set it before init, or unset it |
| `venv` fails with ensurepip error (Debian/Ubuntu) | `python3-venv` not installed | ask, then `sudo apt-get install -y python3-venv`, or `--without-pip` |
| Test suite: one cookie test fails on a fresh resolve | fixed in current main; older clones pin `-c constraints.txt` | pull, or install with the constraints file |

## Uninstall

```
<venv-python> -m atlas_plus.uninstall_cmd --all --yes
```
removes the corpus and state after a report. Then `claude mcp remove sextant` and delete the
clone. Nothing else was installed.
