# CanLab — Technical Specification

> **Version**: 1.0 (Alpha)  
> **Date**: 2026-09-10  
> **Status**: Active development

---

## 1. Overview

CanLab is a desktop application for reverse-engineering CAN bus data. It provides tools for capturing, analyzing, decoding, and (on isolated bench setups) injecting CAN frames. The application is built with Python 3.11+ and PyQt6.

### 1.1 Goals

- Provide a unified workstation for CAN bus reverse engineering
- Support multiple CAN interfaces via python-can
- Automate signal discovery and DBC generation
- Integrate AI-assisted signal interpretation
- Maintain strict safety boundaries between analysis and transmission

### 1.2 Non-Goals

- Real-time ECU flashing or calibration
- Vehicle-specific tuning (use dedicated tools)
- Production-grade automotive diagnostics (use OEM tools)

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────┐
│           PyQt6 GUI Layer               │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────────┐  │
│  │Tabs │ │Panel│ │Panel│ │Settings │  │
│  │(15) │ │ ID  │ │Insp.│ │ Dialog  │  │
│  └──┬──┘ └──┬──┘ └──┬──┘ └────┬────┘  │
│     └────────┴────────┴────────┘       │
│              MainWindow                 │
│              (QMainWindow)              │
└─────────────────┬───────────────────────┘
                  │ signals/slots
┌─────────────────▼───────────────────────┐
│           AppState (Singleton)          │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│  │ Frames  │ │ Signals │ │   DBC   │  │
│  │DataFrame│ │  list   │ │ manager │  │
│  └─────────┘ └─────────┘ └─────────┘  │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│            Core Engine Layer            │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│  │Analysis │ │Protocol │ │ Safety  │  │
│  │(40+ mod)│ │(UDS/J1939│ │  Gate   │  │
│  │         │ │/OBD-II) │ │         │  │
│  └─────────┘ └─────────┘ └─────────┘  │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         Hardware Abstraction            │
│         python-can (BusABC)             │
│    ┌────────┐ ┌────────┐ ┌────────┐   │
│    │ Socket │ │  PCAN  │ │ Vector │   │
│    │  CAN   │ │        │ │        │   │
│    └────────┘ └────────┘ └────────┘   │
└─────────────────────────────────────────┘
```

### 2.2 Threading Model

| Thread | Purpose | Lifetime |
|--------|---------|----------|
| Main (GUI) | Event loop, rendering, user input | Application lifetime |
| LiveCANWorker | Real-time CAN frame capture | While "Live" mode active |
| MultiBusWorker | Multi-interface capture | While multi-bus active |
| ReplayWorker | Log file replay | While replaying |
| InjectionWorker | Frame injection | While injecting |
| FuzzerWorker | Fuzzing campaigns | While fuzzing |
| GatewayWorker | MitM forwarding | While gateway active |
| SafetyScanWorker | Safety-critical sweeps | While scanning |
| UDSScanWorker | Diagnostic scans | While scanning |
| SecurityAccessWorker | Seed/key cracking | While cracking |
| REST API Thread | Local HTTP server | While API enabled |
| ComputeWorker | Background analysis | On demand |

**Rule**: All workers must implement `stop()` and respect `self._running`. MainWindow's `closeEvent` calls `_stop_tab_workers()` to join all threads before exit.

---

## 3. Data Model

### 3.1 Frame Storage (`core/state.py`)

```python
class AppState(QObject):
    # Signals
    frames_updated = pyqtSignal()
    id_selected = pyqtSignal(str)
    dbc_updated = pyqtSignal()
    # ... 20+ signals

    # Storage
    _frames_base: pd.DataFrame      # Historical frames (from files)
    _frame_chunks: list[pd.DataFrame] # Live chunks (appended)
    _frames_cache: pd.DataFrame     # Lazy concatenation cache
    max_frames: int = 500_000       # Memory cap
```

**Frame DataFrame schema**:

| Column | Type | Description |
|--------|------|-------------|
| `Timestamp` | float | Seconds since epoch |
| `ID` | str | Hex CAN ID (e.g., "0x123") |
| `DLC` | int | Data length code (0-8) |
| `B0`-`B7` | int | Data bytes |
| `Extended` | bool | 29-bit ID flag |
| `Bus` | str | Interface name (multi-bus) |
| `Direction` | str | "RX" or "TX" |

### 3.2 Signal Definition

```python
signal_def = {
    "message_id": "0x123",
    "message_name": "WHL_SPD",
    "signal_name": "WHL_SPD_FL",
    "start_bit": 0,
    "length": 16,
    "byte_order": "little",  # or "big"
    "value_type": "unsigned",  # or "signed"
    "scale": 0.03125,
    "offset": 0.0,
    "minimum": 0.0,
    "maximum": 255.996875,
    "unit": "km/h",
}
```

### 3.3 DBC Management

- Uses `cantools` for DBC parsing, encoding, decoding
- Supports DBC, ARXML, CAN matrix import
- Exports: DBC, openpilot DBC, CANdb++, ARXML, Wireshark Lua

---

## 4. Safety Architecture

### 4.1 ARM TX Gate (`core/safety.py`)

```python
_armed = False  # Global state, disarmed by default

def require_armed() -> None:
    """Raise BusNotArmedError unless TX is explicitly armed."""
    if not is_armed():
        raise BusNotArmedError("Bus transmit is disarmed...")
```

**Protected paths** (must call `require_armed()` before every `bus.send()`):

| Module | Method | Purpose |
|--------|--------|---------|
| `injection.py` | `InjectionWorker.run()` | Signal injection |
| `fuzzer.py` | `FuzzerWorker.run()` | Fuzzing campaigns |
| `replay.py` | `ReplayWorker.run()` | Log replay |
| `gateway.py` | `GatewayWorker.run()` | MitM forwarding |
| `safety_scanner.py` | `SafetyScanWorker.run()` | Safety sweeps |
| `uds.py` | `UDSScanner._send_to()` | UDS requests |
| `uds.py` | `UDSScanner._send_and_recv()` | UDS requests |
| `rest_api.py` | `/inject` endpoint | REST injection |

### 4.2 Safety Features

- **First-launch disclaimer**: Cannot proceed without acknowledging risks
- **ARM TX toggle**: Toolbar button, disarmed by default, requires explicit click
- **Read-only UDS scan**: Destructive services require separate checkbox + confirmation
- **Path whitelist**: Cache clear only allowed inside `~/.canlab`
- **Expression sandbox**: Custom seed→key expressions validated by AST whitelist

---

## 5. Protocol Support

### 5.1 CAN / CAN FD

- Classical CAN: 11-bit and 29-bit IDs, up to 8 bytes
- CAN FD: up to 64 bytes, BRS (Bit Rate Switch) support
- Log formats: CSV, candump, PCAN, Vector ASC, MDF (optional)

### 5.2 UDS (ISO 14229)

| Service | ID | Support |
|---------|-----|---------|
| DiagnosticSessionControl | 0x10 | ✅ |
| ECUReset | 0x11 | ✅ |
| SecurityAccess | 0x27 | ✅ |
| ReadDataByIdentifier | 0x22 | ✅ |
| WriteDataByIdentifier | 0x2E | ✅ |
| RoutineControl | 0x31 | ✅ |
| RequestDownload | 0x34 | ✅ |
| TransferData | 0x36 | ✅ |
| RequestTransferExit | 0x37 | ✅ |

### 5.3 J1939

- PGN/SPN decoding
- DM1 (Active Diagnostic Trouble Codes)
- DM2 (Previously Active DTCs)

### 5.4 OBD-II (ISO 15031)

- Mode 01: Current data (PID polling)
- Mode 03: Stored DTCs
- Mode 04: Clear DTCs
- Mode 09: Vehicle information

### 5.5 ISO-TP (ISO 15765-2)

- Single frame (SF)
- First frame (FF)
- Consecutive frame (CF)
- Flow control (FC)

### 5.6 XCP (Universal Measurement and Calibration Protocol)

- Basic XCP commands
- DAQ list configuration

### 5.7 DoIP (ISO 13400)

- Diagnostics over IP
- Vehicle discovery
- Routing activation

---

## 6. AI Integration

### 6.1 Providers

| Provider | Model | API Key | Local |
|----------|-------|---------|-------|
| Anthropic | Claude 3.5 Sonnet | Required | No |
| Groq | Llama 3.1 70B | Required | No |
| Ollama | Any local model | Not required | Yes |

### 6.2 AI Engine Tab

- Send CAN ID + captured frames to AI for interpretation
- Offline ML findings injected into prompt (entropy, periodicity, correlation)
- Persistent conversation memory across sessions
- Markdown rendering of responses

### 6.3 MCP Server

- Model Context Protocol server for external AI tools
- Exposes CanLab state and analysis results
- Runs on localhost, requires explicit enable

---

## 7. Plugin System

### 7.1 Plugin API

```python
# examples/plugins/hello_plugin.py
def register(api):
    """Called when plugin is loaded."""
    api.add_menu_action("Hello", on_hello)
    api.add_frame_handler(on_frame)  # Optional: per-frame callback

def on_hello(api):
    api.show_message("Hello from plugin!")

def on_frame(api, frame):
    pass  # Process each frame
```

### 7.2 Plugin Discovery

- Scans `plugins/` directory at startup
- Loads `.py` files with `register()` function
- Sandboxed: plugins run in main thread, can access AppState

---

## 8. REST API

### 8.1 Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/status` | Server status, frame count, armed state |
| GET | `/frames` | Recent frames (paginated) |
| GET | `/frames/{id}` | Frames for specific CAN ID |
| POST | `/inject` | Inject frame (requires ARM TX + token) |
| GET | `/signals` | Current signal values |
| GET | `/dbc` | Current DBC as JSON |

### 8.2 Authentication

- Bearer token in `Authorization` header
- Token generated on first API enable
- Stored in system keyring

---

## 9. Testing

### 9.1 Test Structure

```
tests/
├── conftest.py              # Fixtures (mock bus, sample frames)
├── test_audit_fixes.py      # Regression tests for fixed bugs
├── test_safety.py           # Safety gate tests
├── test_log_importers.py    # Log format tests
├── test_rest_api.py         # API endpoint tests
├── test_uds_safety.py       # UDS safety tests
└── ...                      # 20+ test files
```

### 9.2 Test Coverage

| Category | Files | Coverage |
|----------|-------|----------|
| Core logic | 15 | Good |
| Safety | 2 | Good |
| Log formats | 1 | Good |
| REST API | 1 | Basic |
| UI | 0 | **None** |

### 9.3 Running Tests

```bash
pytest tests/ -q                    # All tests
pytest tests/test_safety.py -v      # Specific file
pytest -k "test_arm" -v             # Pattern match
```

**Expected**: 151 passed, 1 skipped (MDF requires `asammdf`)

---

## 10. Build & Distribution

### 10.1 PyInstaller

```bash
# Build standalone executable
pyinstaller canlab.spec

# Output in dist/CanLab/
```

### 10.2 PyInstaller Spec (`canlab.spec`)

- Single-file or one-dir mode
- Bundles Qt plugins, cantools, pandas
- Icon: `canlab/canlab.png`
- Version info embedded

---

## 11. Performance Targets

| Metric | Target | Current |
|--------|--------|---------|
| Frame capture rate | 10,000 fps | ~5,000 fps |
| Memory (1M frames) | < 500 MB | ~400 MB |
| Startup time | < 3 s | ~2 s |
| Plot refresh (10 signals) | 60 fps | ~30 fps |
| Log file load (1M frames) | < 10 s | ~5 s |

---

## 12. Security Considerations

### 12.1 Threat Model

| Threat | Mitigation |
|--------|------------|
| Accidental injection on live bus | ARM TX gate, disarmed by default |
| Malicious log file exploitation | Input validation, no code execution |
| API token theft | System keyring storage |
| Expression injection | AST whitelist, no `__builtins__` |
| Path traversal in cache clear | Whitelist to `~/.canlab` |

### 12.2 Data Privacy

- No telemetry or analytics
- API keys stored in system keyring (not plaintext)
- Conversation history stored locally (`~/.canlab/`)
- No cloud sync without explicit user action

---

## 13. Future Roadmap

### Short-term (next release)

- [ ] UI test coverage with pytest-qt
- [ ] Fix remaining MI normalization bug
- [ ] Fix checksum detection direction
- [ ] Gateway queue timeout handling
- [ ] Plot tab performance optimization

### Medium-term

- [ ] CAN FD full support (all tabs)
- [ ] Ethernet/DoIP capture
- [ ] Cloud DBC sharing (opt-in)
- [ ] Collaborative analysis sessions

### Long-term

- [ ] Web-based version (Pyodide/WebAssembly)
- [ ] Mobile companion app
- [ ] Hardware-in-the-loop integration

---

## 14. Appendix

### 14.1 File Format Support Matrix

| Format | Import | Export | Notes |
|--------|--------|--------|-------|
| DBC | ✅ | ✅ | Primary format |
| ARXML | ✅ | ✅ | Experimental export |
| CAN matrix | ✅ | ❌ | CSV-like format |
| CANdb++ | ❌ | ✅ | Export only |
| openpilot DBC | ❌ | ✅ | Export only |
| Wireshark Lua | ❌ | ✅ | Export only |
| CSV | ✅ | ✅ | Generic frame log |
| candump | ✅ | ✅ | Linux can-utils |
| PCAN | ✅ | ❌ | PEAK-System |
| Vector ASC | ✅ | ❌ | Vector tools |
| MDF | ✅ | ❌ | Requires `asammdf` |

### 14.2 CAN Interface Support

| Interface | Windows | Linux | macOS |
|-----------|---------|-------|-------|
| SocketCAN | ❌ | ✅ | ❌ |
| PCAN | ✅ | ✅ | ❌ |
| Vector | ✅ | ❌ | ❌ |
| Kvaser | ✅ | ✅ | ❌ |
| SLCAN | ✅ | ✅ | ✅ |
| Virtual | ✅ | ✅ | ✅ |

### 14.3 Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| Ctrl+O | Open log file |
| Ctrl+S | Save DBC |
| Ctrl+Q | Quit |
| F5 | Refresh frames |
| Ctrl+F | Find ID |
| Space | Freeze/follow |

---

*End of specification*
