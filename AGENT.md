# CanLab — Agent Guide

> **Purpose**: This document tells AI agents (Claude, Copilot, etc.) how to understand, build, test, and contribute to this codebase. It is the single source of truth for project conventions, architecture decisions, and development workflow.

---

## Project Identity

- **Name**: CanLab — CAN Bus Reverse-Engineering Workstation
- **Language**: Python 3.11+
- **GUI Framework**: PyQt6
- **License**: MIT
- **Repository**: https://github.com/yjw17694927050-art/CAN
- **Status**: Alpha — actively developed, single-author project

---

## Architecture Overview

```
canlab/
├── main.py                 # Entry point: QApplication, safety disclaimer, MainWindow
├── mainwindow.py           # Main window, toolbar, tab container, REST API lifecycle
├── mcp_server.py           # MCP server for AI tool integration
├── settings_dialog.py      # Settings UI (API keys, cache, GitHub token)
├── theme.py                # QSS stylesheet, fonts, colors
│
├── core/                   # Business logic (no GUI imports allowed)
│   ├── state.py            # AppState singleton: frames, signals, DBC, memory cap
│   ├── safety.py           # Global ARM TX gate — every TX path must call require_armed()
│   ├── log_parser.py       # Log import (CSV, candump, PCAN, Vector, MDF)
│   ├── dbc_manager.py      # DBC load/save/validate via cantools
│   ├── signal_analyzer.py  # Entropy, periodicity, suspected type classification
│   ├── auto_dbc.py         # Auto-generate DBC from captured frames
│   ├── replay.py           # Log replay with loop, seek, scrubber
│   ├── injection.py        # Frame injection worker
│   ├── fuzzer.py           # Fuzzing worker
│   ├── gateway.py          # MitM gateway between two CAN interfaces
│   ├── safety_scanner.py   # Safety-critical signal sweep (brake, steer, accel)
│   ├── uds.py              # UDS diagnostic scanner
│   ├── security_access.py  # UDS SecurityAccess (0x27) seed→key cracker
│   ├── isotp.py            # ISO-TP session layer
│   ├── j1939.py            # J1939 PGN/SPN decoder
│   ├── obd2_poller.py      # OBD-II Mode 01 PID polling
│   ├── xcp.py              # XCP protocol
│   ├── doip.py             # DoIP (Diagnostics over IP)
│   ├── canfd.py            # CAN FD helpers
│   ├── canid.py            # CAN ID normalization utilities
│   ├── rest_api.py         # Local REST API server
│   ├── plugin_loader.py    # Plugin system
│   └── ...                 # (40+ total core modules)
│
├── tabs/                   # GUI tabs (one per major feature)
│   ├── frames_tab.py       # Raw frame table
│   ├── signals_tab.py      # Decoded signal table
│   ├── plot_tab.py         # Time-series plots
│   ├── ai_engine_tab.py    # AI chat for signal interpretation
│   ├── dbc_builder_tab.py  # Visual DBC editor
│   ├── code_gen_tab.py     # Code generation from DBC
│   ├── intelligence_tab.py # Cross-ID correlation, fingerprint
│   ├── injection_tab.py    # Injection, fuzzer, trigger, replay UI
│   ├── diagnostics_tab.py  # UDS, ISO-TP, J1939, OBD-II UI
│   ├── dashboard_tab.py    # Heatmap, timeline, gauges
│   ├── auto_re_tab.py      # Auto reverse-engineering UI
│   ├── timeline_tab.py     # Event timeline + video sync
│   ├── obd_dashboard_tab.py# Live PID gauges
│   ├── signal_intelligence_tab.py  # ML-based signal classification
│   └── gateway_tab.py      # MitM gateway UI
│
├── panels/                 # Dockable side panels
│   ├── id_panel.py         # ID list with filters
│   └── inspector_panel.py  # Frame inspector
│
├── ui/                     # Reusable UI components
│   ├── animations.py       # PulsingDot, CountUpLabel
│   └── compute_worker.py   # Background compute worker
│
├── tests/                  # pytest test suite
│   ├── test_audit_fixes.py # Tests for previously fixed bugs
│   ├── test_safety.py      # Safety gate tests
│   ├── test_log_importers.py
│   ├── test_rest_api.py
│   └── ...                 # (20+ test files)
│
├── docs/                   # Documentation and screenshots
├── sample_data/            # Example CAN logs for testing
└── examples/               # Plugin examples
```

---

## Critical Safety Rules

These are **non-negotiable**. Any code that violates these will be rejected.

### 1. ARM TX Gate (`core/safety.py`)

**Every** code path that transmits frames onto a CAN bus must call `require_armed()` before `bus.send()`.

```python
from core.safety import require_armed, BusNotArmedError

# ✅ Correct
try:
    require_armed()
except BusNotArmedError:
    self.error.emit("Bus TX is disarmed")
    return
bus.send(msg)

# ❌ Wrong — bypasses the safety gate
bus.send(msg)
```

**Protected paths**: injection, fuzzer, replay, gateway forwarding, safety scanner, UDS scan, REST `/inject`.

### 2. Thread Safety

- All CAN bus I/O runs in `QThread` workers, never on the GUI thread.
- Workers must implement `stop()` and respect `self._running`.
- MainWindow's `closeEvent` stops all tab workers via `_stop_tab_workers()`.

### 3. Memory Safety

- `AppState.max_frames = 500_000` — hard cap on in-memory frames.
- `append_frames()` evicts oldest chunks when the cap is exceeded.

---

## Development Workflow

### Setup

```bash
git clone https://github.com/yjw17694927050-art/CAN.git
cd CAN
python3 -m venv .venv
# Windows
.venv\Scripts\activate
# macOS/Linux
source .venv/bin/activate

pip install -r requirements.txt
```

### Run

```bash
cd canlab
python main.py
```

### Test

```bash
# From repo root
pytest tests/ -q
```

**Expected**: 151 passed, 1 skipped (MDF test requires optional `asammdf`).

### Code Style

- **No GUI imports in `core/`**. Core modules must be testable headless.
- **Signals for cross-thread communication**. Never call GUI methods directly from workers.
- **Type hints** on all public APIs.
- **Docstrings** for all public functions/classes.

---

## Key Design Decisions

### Why PyQt6?

- Native look and feel on all platforms
- Mature threading model (`QThread`, signals/slots)
- Rich widget ecosystem for tables, plots, trees

### Why cantools for DBC?

- Industry standard (used by openpilot, comma.ai)
- Handles DBC parsing, encoding, decoding, validation
- Supports CAN FD

### Why the ARM TX gate pattern?

- Prevents accidental transmission on live vehicle buses
- Single point of control for all TX paths
- Explicit user action required (toolbar toggle)

### Why chunk-based frame storage in AppState?

- O(chunk) append cost instead of O(n) DataFrame concat
- Lazy rebuild of full DataFrame only when read
- Memory cap prevents OOM on long captures

---

## Common Tasks

### Adding a new tab

1. Create `tabs/my_tab.py` inheriting `QWidget`
2. Import in `mainwindow.py` and add to `_add_tabs()`
3. If the tab has background workers, add their attribute names to `_stop_tab_workers()`
4. Write tests in `tests/test_my_tab.py`

### Adding a new core module

1. Create `core/my_module.py`
2. No GUI imports — only `core.*`, `pandas`, `python-can`, etc.
3. If it transmits frames, call `require_armed()` before every `bus.send()`
4. Write tests in `tests/test_my_module.py`

### Adding a new log format

1. Add parser function in `core/log_parser.py`
2. Register in `parse_log_file()` dispatcher
3. Add test file in `tests/test_log_importers.py`

### Adding a new AI provider

1. Add client in `core/ai_client.py`
2. Add key management in `settings_dialog.py`
3. Update `AIEngineTab` to include the new provider

---

## Known Issues & TODOs

See `docs/AUDIT_FIXES.md` for the complete audit history.

### Fixed in latest commit (ff2becb)

| Bug | File | Fix |
|-----|------|-----|
| Replay loop only replays first 8 frames | `core/replay.py` | Renamed shadowed `n` to `dlc_n` |
| Auto-DBC generates wrong signals | `core/auto_dbc.py` | Use real `analyze_id()` keys |
| SecurityAccess subfunction wrong for even levels | `core/security_access.py` | Added parentheses for `&` precedence |
| Safety scanner bypasses ARM TX | `core/safety_scanner.py` | Added `require_armed()` |
| UDS scan bypasses ARM TX | `core/uds.py` | Added `require_armed()` |
| Custom expression eval is escapable | `core/security_access.py` | AST-whitelisted safe compiler |
| Cache clear deletes arbitrary directories | `settings_dialog.py` | Added path whitelist + confirmation |
| Window close crashes with running threads | `mainwindow.py` | Added `_stop_tab_workers()` |
| FD frames silently parsed as empty | `core/log_parser.py` | Full-file FD detection |
| Memory grows unbounded on long captures | `core/state.py` | Added `max_frames` cap |

### Remaining known issues

- **UI thread blocking**: Plot tab recalculates all signals every 50 frames (O(N×signals))
- **MI normalization**: Signal intelligence MI score uses wrong log base (systematic 30% underestimation)
- **Checksum detection direction**: High-entropy bytes incorrectly classified as checksums
- **Gateway queue blocking**: `q.put()` without timeout can hang reader thread
- **No UI tests**: Zero test coverage for tabs, workers, or GUI interactions

---

## Dependencies

### Required

- `python-can` — CAN interface abstraction
- `cantools` — DBC parsing/encoding
- `pandas` — DataFrame operations
- `PyQt6` — GUI framework
- `numpy` — Numerical operations
- `scipy` — Signal processing (correlation, entropy)

### Optional

- `asammdf` — MDF file support (skip test if missing)
- `anthropic` — Claude AI
- `groq` — Groq AI
- `ollama` — Local AI (no API key needed)
- `matplotlib` — Plotting
- `pyqtgraph` — Fast plotting

---

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Write tests for your changes
4. Run `pytest tests/ -q` — all tests must pass
5. Commit with a descriptive message
6. Push and open a Pull Request

### Commit message format

```
<type>: <short description>

<optional longer description>

<optional footer>
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

---

## Contact

- **Author**: YJW (yjw17694927050@gmail.com)
- **Repository**: https://github.com/yjw17694927050-art/CAN
