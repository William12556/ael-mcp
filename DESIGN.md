# ael-mcp Design

---

## Table of Contents

- [1.0 Overview](<#1.0 overview>)
- [2.0 Architecture](<#2.0 architecture>)
- [3.0 Tool Surface](<#3.0 tool surface>)
- [4.0 Process Model](<#4.0 process model>)
- [5.0 File Structure](<#5.0 file structure>)
- [6.0 Configuration](<#6.0 configuration>)
- [7.0 Constraints and Assumptions](<#7.0 constraints and assumptions>)
- [8.0 Out of Scope](<#8.0 out of scope>)
- [Version History](<#version history>)

---

## 1.0 Overview

`ael-mcp` is a standalone MCP (Model Context Protocol) server. It exposes tools that allow a strategic domain LLM client (e.g. Claude Desktop) to launch and manage the AEL (Autonomous Execution Layer) tactical domain.

The tactical domain is implemented by `orchestrator.py` from the [LLM-Governance-and-Orchestration](https://github.com/William12556/LLM-Governance-and-Orchestration) framework. `ael-mcp` invokes it as an external subprocess. No dependency on framework internals is introduced.

[Return to Table of Contents](<#table of contents>)

---

## 2.0 Architecture

### 2.1 Placement

`ael-mcp` is a standalone repository, independent of the framework and all downstream projects. It is registered once in the Claude Desktop MCP configuration and serves all downstream projects via parameterised tool calls.

```
Claude Desktop (strategic domain)
        │
        │  MCP stdio transport
        ▼
    ael-mcp / server.py
        │
        │  subprocess
        ▼
    <project_dir>/ai/ael/src/orchestrator.py
```

### 2.2 Transport

stdio — the standard transport for locally registered Claude Desktop MCP servers. No port, no auth, no network dependency.

### 2.3 Project Binding

All tools accept `project_dir` as a required parameter (absolute path). The server uses `project_dir` to:

- Locate `orchestrator.py` at `<project_dir>/ai/ael/src/orchestrator.py`
- Locate `config.yaml` at `<project_dir>/ai/ael/config.yaml`
- Read and write run state at `<project_dir>/.ael/ralph/`

[Return to Table of Contents](<#table of contents>)

---

## 3.0 Tool Surface

### 3.1 `start_ael`

Launches `orchestrator.py` as a detached background process.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `project_dir` | string | yes | Absolute path to the downstream project root |
| `mode` | string | yes | `worker`, `reviewer`, or `loop` |
| `task` | string | yes | Task string or absolute path to a prompt file |

**Returns:** JSON object containing `run_id` (UUID4), `pid` (int), `log_path` (string).

**Behaviour:**
- Validates `project_dir`, `orchestrator.py`, and `config.yaml` exist before spawning.
- Spawns orchestrator as a detached subprocess (stdout/stderr redirected to a log file in `.ael/ralph/`).
- Writes a run record (`mcp-run.json`) to `<project_dir>/.ael/ralph/` containing `run_id`, `pid`, `mode`, `task`, and `started_at`.
- Returns immediately; does not wait for orchestrator completion.

### 3.2 `ael_status`

Reports current AEL run state for a project.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `project_dir` | string | yes | Absolute path to the downstream project root |

**Returns:** JSON object containing:
- `run_id`, `pid`, `mode`, `task`, `started_at` — from `mcp-run.json` if present
- `pid_alive` — boolean, whether the recorded PID is still running
- `state_files` — list of state files present in `.ael/ralph/`
- `shipped` — boolean, whether `.ralph-complete` exists
- `blocked` — boolean, whether `RALPH-BLOCKED.md` exists

### 3.3 `reset_ael`

Invokes `orchestrator.py --mode reset` synchronously.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `project_dir` | string | yes | Absolute path to the downstream project root |

**Returns:** JSON object containing `returncode` (int) and `output` (string).

**Behaviour:**
- Runs synchronously (reset is fast; no detach required).
- Removes `mcp-run.json` after successful reset.

[Return to Table of Contents](<#table of contents>)

---

## 4.0 Process Model

### 4.1 Fire-and-Detach

`start_ael` uses `subprocess.Popen` with the process detached from the MCP server's process group. On macOS/Linux this is achieved via `start_new_session=True`. The MCP server process exiting does not terminate the orchestrator.

### 4.2 Python Interpreter

The orchestrator requires a Python environment with its own dependencies (`openai`, `rich`, `pyyaml`, etc.). The interpreter used to launch `orchestrator.py` is resolved in priority order:

1. `python_bin` field in `mcp-run.json` from a prior run (not applicable to `start_ael`).
2. `python3` on `PATH` as visible to the MCP server process.

The caller (Claude Desktop) is responsible for ensuring the Python environment containing orchestrator dependencies is on `PATH`, or for passing the full interpreter path as a future extension.

**Note:** A future `python_bin` optional parameter on `start_ael` may be added by consensus to allow explicit interpreter specification without relying on `PATH`.

### 4.3 Run Record

`mcp-run.json` schema:

```json
{
  "run_id": "uuid4",
  "pid": 12345,
  "mode": "loop",
  "task": "implement the login module",
  "started_at": "2026-04-25T07:00:00",
  "log_path": "/path/to/.ael/ralph/mcp-12345.log"
}
```

[Return to Table of Contents](<#table of contents>)

---

## 5.0 File Structure

```
ael-mcp/
├── server.py          # MCP server — single file
├── requirements.txt   # mcp package only
├── DESIGN.md
├── README.md
├── LICENSE
└── .gitignore
```

[Return to Table of Contents](<#table of contents>)

---

## 6.0 Configuration

### 6.1 Claude Desktop Registration

Add to `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "ael-mcp": {
      "command": "/usr/bin/python3",
      "args": ["/Users/williamwatson/Documents/GitHub/ael-mcp/server.py"]
    }
  }
}
```

Replace `/usr/bin/python3` with the Python interpreter that has the `mcp` package installed.

### 6.2 Dependencies

`server.py` requires only the `mcp` Python package. All other imports are stdlib (`asyncio`, `subprocess`, `json`, `os`, `pathlib`, `uuid`, `datetime`).

[Return to Table of Contents](<#table of contents>)

---

## 7.0 Constraints and Assumptions

- `orchestrator.py` is located at `<project_dir>/ai/ael/src/orchestrator.py`. This path is a framework convention.
- `config.yaml` is located at `<project_dir>/ai/ael/config.yaml`. This path is a framework convention.
- The Python interpreter used to spawn the orchestrator must have the orchestrator's runtime dependencies installed.
- Only one active AEL run per project is tracked via `mcp-run.json`. Concurrent runs within the same project are not supported.
- `ael_status` uses `os.kill(pid, 0)` to check if the recorded PID is alive. On macOS/Linux this is reliable for processes owned by the same user.

[Return to Table of Contents](<#table of contents>)

---

## 8.0 Out of Scope

- Streaming orchestrator output to the MCP client.
- Per-project MCP server registration.
- Authentication or remote transport.
- Concurrent run management.
- Windows support.

[Return to Table of Contents](<#table of contents>)

---

## Version History

| Version | Date | Description |
|---|---|---|
| 0.1 | 2026-04-25 | Initial design |

---

Copyright (c) 2026 William Watson. MIT License.
