# ael-mcp

MCP server for launching and managing the AEL (Autonomous Execution Layer) tactical domain from a strategic domain LLM client such as Claude Desktop.

## Overview

Exposes three tools to an MCP client:

| Tool | Description |
|---|---|
| `start_ael` | Launch `orchestrator.py` in detached mode for a given project |
| `ael_status` | Report current run state from the project's `.ael/ralph/` directory |
| `reset_ael` | Invoke `orchestrator.py --mode reset` to clear AEL state |

## Requirements

- Python 3.11+ with the [`mcp`](https://pypi.org/project/mcp/) package
- A downstream project using the [LLM-Governance-and-Orchestration](https://github.com/William12556/LLM-Governance-and-Orchestration) framework

## Installation

```bash
git clone https://github.com/William12556/ael-mcp.git
```

No additional package installation is required if the `mcp` package is already present in the target Python interpreter (e.g. `/opt/homebrew/bin/python3`).

To verify:
```bash
/opt/homebrew/bin/python3 -m pip show mcp
```

If not present:
```bash
/opt/homebrew/bin/pip3 install mcp
```

## Claude Desktop Configuration

Add to `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "ael-mcp": {
      "command": "/opt/homebrew/bin/python3",
      "args": ["/Users/williamwatson/Documents/GitHub/ael-mcp/server.py"]
    }
  }
}
```

## Tool Reference

### `start_ael`

Launches `orchestrator.py` as a detached background process.

| Parameter | Type | Description |
|---|---|---|
| `project_dir` | string | Absolute path to the downstream project root |
| `mode` | string | `worker`, `reviewer`, or `loop` |
| `task` | string | Task description or path to a prompt file |

Returns: `run_id` (UUID), PID, log path.

### `ael_status`

Reads `.ael/ralph/` state files from the project directory.

| Parameter | Type | Description |
|---|---|---|
| `project_dir` | string | Absolute path to the downstream project root |

Returns: run state summary (active PID if running, state files present).

### `reset_ael`

Invokes `orchestrator.py --mode reset` synchronously.

| Parameter | Type | Description |
|---|---|---|
| `project_dir` | string | Absolute path to the downstream project root |

## License

Copyright (c) 2026 William Watson. MIT License.
