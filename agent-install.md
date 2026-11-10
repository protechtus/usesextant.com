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

Run whichever of these exist and record the output (one per line; do not chain them with
`||`, PowerShell rejects that):

```
python --version
python3 --version
py -3 --version
git --version
```

- Sextant needs **Python 3.11 or newer** and **git**. Check every interpreter that answered,
  not just the first: on many machines `python` is an old 3.10 while `py -3` or `python3.12`
  is current. Use the newest 3.11+ you found for every later step. If none qualifies, **ASK**
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

(When the package is on PyPI, `pipx install sextant-ai` or `uv tool install sextant-ai` is
the short path and skips steps 1 to 3. Until then, install from the clone. Never run
`pip install sextant`: that is an unrelated package.)

## 2. Create the virtual environment

Windows PowerShell:

```
py -3 -m venv .venv
.venv\Scripts\python.exe -m pip install -U pip setuptools
```

Windows Git Bash (same layout, forward slashes):

```
py -3 -m venv .venv
.venv/Scripts/python.exe -m pip install -U pip setuptools
```

macOS / Linux / WSL:

```
python3 -m venv .venv
.venv/bin/python -m pip install -U pip setuptools
```

Verify: `.venv/bin/python --version` (or `.venv\Scripts\python.exe --version`) prints 3.11+.

Known failure: on Debian/Ubuntu, `venv` fails with an `ensurepip` error when the
`python3-venv` package is missing. **ASK** before running
`sudo apt-get install -y python3-venv`. If the human cannot use sudo:

```
python3 -m venv --without-pip .venv
curl -sSLo get-pip.py https://bootstrap.pypa.io/get-pip.py
.venv/bin/python get-pip.py
```
or, if `uv` is installed, `uv venv --seed .venv` (the `--seed` flag installs pip so the
`python -m pip` commands below keep working).

## 3. Install

Use the venv's Python explicitly (do not rely on `activate`; it is shell-specific):

```
<venv-python> -m pip install -e ".[dev,telegram,summarizer]"
```

Verify: `<venv-python> -m pip show sextant-ai` prints the version, and
`<venv-python> -m atlas_plus --version` prints it too.

Known failure: a dependency resolver conflict. Retry with the pinned set:
`<venv-python> -m pip install -e ".[dev,telegram,summarizer]" -c constraints.txt`.

## 4. Initialise the memory corpus

**ASK first:** by default this writes two folders outside the project, `~/.memory` (the
markdown corpus the human owns) and `~/.atlas` (state). Ask where they want them. To keep
everything inside the install folder instead, set the variables **before** running init and
keep them set for every later command (and pass them to the MCP client in step 5):

```
ATLAS_MEMORY_ROOT=<install-dir>/memory
ATLAS_STATE_DIR=<install-dir>/state
```
(`ATLAS_WORKSPACE_ROOT=<dir>` re-roots both at once as `<dir>/.memory` and `<dir>/.atlas`.)

Then:

```
<venv-python> -m atlas_plus.init_cmd
```

Verify: `<venv-python> -m atlas_plus.doctor.cli` runs and prints the corpus path near the
top. On a fresh install the Services rows (Brain, MCP, Ollama) show **down**: nothing is
running yet, which is expected and not an error.

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

If you set `ATLAS_MEMORY_ROOT`/`ATLAS_STATE_DIR` (or `ATLAS_WORKSPACE_ROOT`) in step 4, pass
the same variables **after the server name**, one `-e` each:
`claude mcp add -s user sextant -e ATLAS_WORKSPACE_ROOT=/their/path -- <command>`. Putting
`-e` before the name fails on current Claude Code.

WSL install, Windows Claude Code: the command must cross into WSL, so register
`wsl -d <Distro> -- /home/<user>/sextant/.venv/bin/python -m atlas_plus.mcp.memory_server.server --stdio`
(with the Linux path to the venv).

Verify: `claude mcp list` shows `sextant ... Connected`.

**Claude Desktop**: merge `deploy/claude_desktop_config.snippet.json` into the client's
config file (`%APPDATA%\Claude\claude_desktop_config.json` on Windows,
`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS), replacing the
placeholder path with the venv Python. **ASK** before editing that file; show the diff.

**Cursor and other MCP clients**: same command and arguments as above, in the client's MCP
settings. The server name `sextant` must be unique per machine; a second install needs a
different name.

**Session-start priming for Claude Code** (optional, recommended): hook snippets live in
`deploy/`. Pick the pair for the platform: `session_start_hook.json` and `stop_hook.json`
on Windows (they call `.venv\Scripts\python.exe`), `session_start_hook.macos.json` and
`stop_hook.macos.json` on macOS **and Linux** (they call `.venv/bin/python`; the file name
is historical). They add hooks to `~/.claude/settings.json`. **ASK** before editing settings,
show the diff, and if a `hooks` key already exists add the entries rather than replacing the
key. Hooks take effect in the next Claude Code session.

Hooks are separate processes: the `env` block of the MCP registration applies to the MCP
server only, not to them. If you set `ATLAS_MEMORY_ROOT` / `ATLAS_STATE_DIR` /
`ATLAS_PROJECTS_ROOT` in the registration rather than in the user's environment, the hook
would read a different corpus. Either set the same variables user-wide (`setx` on Windows,
the shell profile on POSIX) or put them on the hook command itself
(`ATLAS_MEMORY_ROOT=/path python -m atlas_plus.hooks.session_start` on POSIX; a one-line
`.cmd` wrapper on Windows). The round-trip in step 6 plus a session-start run from a
project directory should show the same identity note from both.

## 6. Prove it works (the round-trip)

The shortest path, from the clone, with the same environment variables as step 4:

```
<venv-python> scripts/mcp_roundtrip.py --project demo
```

It starts the server over stdio, does the MCP handshake, and runs the three calls below,
printing each result as JSON and ending with `ROUND-TRIP: OK`. Paste that output into your
report.

If the human prefers to see it through their connected client, do the same three calls there.
Exact call shapes (parameter names matter):

1. `memory_health` with no arguments. Expect `"status": "ok"`.
2. `memory_remember_tool` with `scope="project:demo"` and
   `fact="Sextant installed by an agent on <date>."`. Expect a result naming a file under
   `projects/`.
3. `memory_search_tool` with `query="Sextant installed by an agent"`. Expect the fact back
   with a `confidence` band (high/medium/low/starved).

Scope grammar for call 2: `project:<name>`, `reference:<topic>`, `preference:<topic>`. A bare
`reference` is rejected.

The CLI can confirm the write too: `<venv-python> -m atlas_plus search "installed by an agent"`.

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

Run the report first; it lists exactly what exists and what still references the install:

```
<venv-python> -m atlas_plus.uninstall_cmd
```

Then, **ASK** before each of these, because the corpus is the human's notes:

1. `<venv-python> -m atlas_plus.uninstall_cmd --all --yes --confirm-path <corpus path> --confirm-state-path <state path>`
   removes the state dir and the corpus. Both paths must be named (the report prints them);
   the state dir is not a cache: it holds captured transcripts, the task database, the
   intake ledger, sync devices and research reports. A corpus containing a `.sextant-protect` marker refuses deletion until the
   human removes the marker by hand. A state dir that overlaps the corpus, home, or workspace
   root is refused outright.
2. `claude mcp remove -s user sextant` (or the equivalent in the client they used), and remove
   any Stop or SessionStart hook entries that point at the clone from `~/.claude/settings.json`.
3. The session log the summarizer wrote (path shown in the report) is never deleted by the
   tool; the human decides.
4. `pip uninstall sextant-ai` inside the venv, or simply delete the clone and its `.venv`.

Use the same `ATLAS_*` environment variables you used at install time, or the report will
describe a different (probably empty) location.
