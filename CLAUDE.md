# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

XiaoZhi (小智) is an ESP-IDF C/C++ voice-assistant firmware for ESP32-family chips. It streams audio over WebSocket or MQTT+UDP to a cloud server that runs ASR/LLM/TTS pipelines. The firmware supports 138+ board directories and 172+ release variants across ESP32, ESP32-C3/C5/C6, ESP32-S3, and ESP32-P4.

## Build Commands

Always source the ESP-IDF environment first. **Use ESP-IDF v6.0.2** (IDF 5.5.x is only for documented legacy boards).

```sh
source /path/to/esp-idf/export.sh
```

```sh
# List all board variants
python3 scripts/release.py --list-boards

# Build one variant (canonical entry point; changes sdkconfig)
python3 scripts/release.py <board-directory> --name <variant-name>

# Build all variants of a board
python3 scripts/release.py <board-directory>

# Host-side tests
python3 -m unittest discover -s scripts/tests -v

# Format files (Google C++ style, 4-space indent, 100-col width)
clang-format -i <file>
clang-format --dry-run -Werror <file>
```

## Architecture

```
main/
├── main.cc                  # Entry point: creates Board, initializes Application
├── application.cc/.h        # Main event loop, protocol lifecycle, state orchestration
├── device_state.cc/.h       # State enum: Unknown→Starting→Idle/Connecting/Listening/Speaking/…
├── device_state_machine.cc/.h  # Validates & enforces legal state transitions
├── mcp_server.cc/.h         # Device-side MCP tool registry and dispatch
├── ota.cc/.h                # OTA firmware update
├── settings.cc/.h           # NVS-backed persistent settings
├── assets.cc/.h             # Asset partition management
├── audio/
│   ├── audio_service.cc/.h  # Dual-task pipeline: MIC→encode→send, recv→decode→speaker
│   ├── audio_codec.h        # Abstract HW codec interface (I2S config, volume, mute)
│   ├── audio_engine.h       # Abstract wake-word/VAD engine (AFE for S3/P4, lite for others)
│   ├── codecs/              # Per-codec implementations (ES8311/8374/8388/8389, box, dummy)
│   ├── engines/             # afe_audio_engine (S3/P4) and lite_audio_engine (C3/C5/C6)
│   ├── wake_words/          # Custom wake word + ESP-SR built-in wake words
│   └── demuxer/             # OGG demuxer for localized audio prompts
├── protocols/
│   ├── protocol.h           # Abstract transport: OpenAudioChannel, SendAudio, SendText, callbacks
│   ├── websocket_protocol.cc/.h  # WebSocket transport
│   └── mqtt_protocol.cc/.h       # MQTT+UDP transport
├── display/                 # OLED (SSD1306), LCD (LVGL-based), emote/animated displays
├── led/                     # Single LED, circular strip, GPIO LED
└── boards/
    ├── common/              # Shared board infrastructure: Board base class, WiFi, Ethernet,
    │                        # ML307/EC801E/NT26 4G, dual-network, battery, button, camera,
    │                        # backlight, knob, PTT MCP tool, sleep/power-save timers
    └── <board-name>/        # One directory per board; contains config.h, board .cc, optional
                             # custom codec/display/PMU implementations
```

**Key pattern**: `Board` is an abstract base class (`main/boards/common/board.h`). Exactly one concrete board class is selected at build time via `DECLARE_BOARD(ClassName)`. All core code depends on `Board` interfaces — never on a concrete board class.

**Audio data flow**: MIC → AudioEngine (wake-word/VAD/AEC) → Opus Encoder → Send Queue → Server → Decode Queue → Opus Decoder → Playback Queue → Speaker. Dedicated FreeRTOS tasks for input, output, and Opus codec.

## Board Selection Chain

Adding a board requires touching every link in this chain:

1. `config.json` (in board directory) — defines target chip, build variants, sdkconfig overrides
2. `scripts/release.py` — discovers boards via `config.json`, drives `idf.py set-target` and build
3. `main/Kconfig.projbuild` — Kconfig `CONFIG_BOARD_TYPE_*` symbol for menuconfig visibility
4. `main/CMakeLists.txt` — maps each `CONFIG_BOARD_TYPE_*` to a `BOARD_TYPE` string, selects fonts and emoji collection
5. Board source files — `config.h`, the `.cc` implementing `Board` with `DECLARE_BOARD(ClassName)`
6. Board documentation — `README.md` in the board directory

Board identity affects OTA compatibility. Never alter an existing board's pins to support different hardware — add a new board or variant.

## Key Rules (see AGENTS.md for full details)

- Change runtime state only through `Application::SetDeviceState()` and `DeviceStateMachine`.
- Callbacks may run outside the main task; schedule mutations with `Application::Schedule()` or event bits.
- Do not block the main event loop or audio tasks; avoid unbounded queues in audio paths.
- Keep shared message semantics in `Protocol`; verify both transports (WebSocket + MQTT/UDP) when changing its contract.
- Validate network input; preserve `cJSON` ownership rules.
- NVS keys are persistent API — they require migration when changed.
- Guard target-specific features with Kconfig/component rules; do not assume every target has PSRAM.
- Treat camera, backlight, display, LED, battery as optional capabilities.
- Format only touched C/C++ files with `.clang-format`; avoid unrelated mass reformatting.
- Do not manually edit: `build/`, `releases/`, `managed_components/`, `components/`, `sdkconfig*`, `main/assets/lang_config.h`, or generated mmap headers.

## Testing and Validation

- Board-only change: build affected variants and smoke-test changed hardware.
- Core/common/audio/protocol/display/Kconfig/CMake change: run host tests (`scripts/tests/`) and build representative affected chip/network paths.
- Protocol changes: verify both WebSocket and MQTT/UDP when shared behavior changes.
- Audio changes: verify capture, playback, wake/VAD, interruption, reconnect, and applicable AEC modes.
- A successful build is not hardware validation — always report what was tested and what still needs physical hardware.

## Authoritative Documentation

- `AGENTS.md` — exhaustive rules, conventions, and subsystem details
- `docs/esp-idf-6-migration.md` — SDK compatibility and migration notes
- `docs/custom-board.md` — custom board creation guide
- `docs/websocket.md` / `docs/mqtt-udp.md` — wire protocol details
- `docs/mcp-protocol.md` / `docs/mcp-usage.md` — MCP device control
- `main/audio/README.md` — audio subsystem design
- `docs/code_style.md` — code formatting guide
- `.github/workflows/build.yml` — CI build matrix
