# esp-idf-mcp-monitor

Serial monitor tools for the ESP-IDF MCP server. Drop-in replacement for `tools/idf_py_actions/mcp_ext.py` that adds interactive monitor support to `idf.py mcp-server`.

**Upstream PR:** [espressif/esp-idf#18385](https://github.com/espressif/esp-idf/pull/18385)

## The Problem

ESP-IDF v6.0 introduced an MCP server (`idf.py mcp-server`) with tools for `build`, `flash`, `set-target`, and `clean`. But there's no way for an AI agent to read serial output from the device — no boot logs, no error messages, no runtime diagnostics.

The agent can push firmware to the board but can't see what happens next.

Two technical barriers prevent simply wrapping `idf.py monitor`:

1. **TTY requirement** — `idf_monitor.py` checks `sys.stdin.isatty()` and refuses to start without a real terminal
2. **Process tree cleanup** — `idf.py` spawns `idf_monitor.py` as a child; killing only the parent leaves the child holding the serial port open

## What This Adds

Five new MCP tools, all using PTY-wrapped `idf.py monitor`:

| Tool | Purpose |
|---|---|
| `monitor_boot` | One-shot: reset board, capture boot log, return it. Supports `wait_for` to stop early on a pattern match. |
| `monitor_start` | Start a persistent background monitor session with board reset. |
| `monitor_send` | Send a command to the device (write-only). |
| `monitor_read` | Poll buffered output. Supports `timeout` and `wait_for` for waiting on responses. |
| `monitor_stop` | Kill the session and clean up. |

Existing tools (`build_project`, `set_target`, `flash_project`, `clean_project`) are unchanged.

### Additional improvements

- `_idf_cmd()` helper eliminates repeated command construction and validates `IDF_PATH`
- PTY echo disabled via `termios` to prevent command reflection in captured output
- ANSI escape stripping so agents get clean text
- Process-group killing (`os.killpg`) to prevent zombie `idf_monitor.py` processes
- Guarded `pty`/`select`/`termios` imports — existing tools still work on Windows

## Installation

Copy the file over the original:

```bash
cp mcp_ext.py $IDF_PATH/tools/idf_py_actions/mcp_ext.py
```

The five monitor tools will appear alongside the existing ones when the MCP server starts.

## Usage with Claude Code

**First-time setup:**

```bash
claude mcp add --transport stdio esp-idf -- idf.py mcp-server
```

**Already connected but need to reload (e.g. after updating the file):**

In Claude Code, type `/mcp` → select `esp-idf` → reconnect.

## Usage

### Quick boot log capture

```
monitor_boot(capture_duration=20)
```

Resets the board, captures 20 seconds of output, returns everything.

### Stop early on a pattern

```
monitor_boot(wait_for="Sensor scan:", capture_duration=30)
```

Returns as soon as "Sensor scan:" appears instead of waiting the full 30 seconds. Useful for unit test results, crash detection, or specific init markers.

### Interactive session

```
monitor_start()
monitor_read(timeout=5)                          # read boot log
monitor_send("STATUS")
monitor_read(wait_for="OK", timeout=3)           # wait for response
monitor_stop()
```

### Example: AI agent debugging

[Read My Blog: I Gave My Claude Access to ESP-IDF Monitor — And It Debugged My PCB](https://jinwookshin.com/playground/i-gave-my-claude-access-to-esp-idf-monitor-and-it-debugged-my-pcb)

## Compatibility

- **ESP-IDF:** v6.0+ (requires MCP server support)
- **Platform:** Monitor tools require Unix-like PTY support (macOS, Linux). Build/flash/clean tools work on all platforms.
- **Tested on:** macOS with ESP32-S3 over USB CDC

## License

Apache-2.0 (same as ESP-IDF)
