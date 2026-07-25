# Repository Guidelines

## Project Overview

XiaoZhi is an ESP-IDF C/C++ voice-assistant firmware supporting 138+ boards and 172+ release variants across ESP32 / C3 / C5 / C6 / S3 / P4 chips. It provides voice interaction via Qwen / DeepSeek models, using MCP protocol for multi-device control. The primary ESP-IDF version is v6.0.2.

## Project Structure & Module Organization

```
main/
├── application.*        —  main event loop, protocol lifecycle, high-level orchestration
├── device_state_machine.* —  legal runtime state transitions (11 states)
├── mcp_server.*         —  device-side MCP tool dispatch (volume, lights, GPIO)
├── ota.* / settings.* / assets.* / system_info.* —  support modules
├── boards/
│   ├── common/          —  Board abstract interface, WiFi/4G/Ethernet network backends
│   ├── <vendor>/<board>/ —  per-board pin maps, config.h, config.json, DECLARE_BOARD
│   └── ...              —  138+ board directories
├── audio/
│   ├── codecs/          —  ES8311 / ES8374 / ES8388 / ES8389 / NoAudio / Dummy / Box
│   ├── engines/         —  AFE (Audio Front-End) and Lite engine wrappers
│   ├── wake_words/      —  Custom + ESP-SR wake word detectors
│   └── demuxer/         —  OGG demuxer for streaming audio
├── protocols/           —  Protocol base class, WebSocket + MQTT/UDP transports
├── display/             —  OLED, LCD, LVGL display backends; emoji, GIF, JPG support
├── led/                 —  Single LED, circular strip (WS2812), GPIO LED patterns
└── assets/              —  locale audio prompts (38 languages), common sound effects
```

```
scripts/
├── release.py           —  canonical build entry point for any board/variant
├── gen_lang.py          —  locale code generator
├── build_default_assets.py —  asset packer
└── tests/               —  host-side Python unit tests
```

## Architecture & Design Patterns

**Component responsibilities:**

| Module | Role |
|---|---|
| `Application` | Singleton; runs the main FreeRTOS event loop; dispatches `MAIN_EVENT_*` bits |
| `DeviceStateMachine` | Enforces legal transitions between `DeviceState` values |
| `Board` | Abstract factory (`create_board()`); exposes audio codec, display, LED, camera, network, battery |
| `Protocol` | Abstract transport; `OnIncomingAudio` / `OnIncomingJson` callback chains |
| `MCP Server` | Registers tool handlers dispatched from cloud-side MCP requests |

**Design rules:**

- Core code depends on `Board` interfaces, never on a concrete board class or board `config.h`.
- Change runtime state only through `Application::SetDeviceState()` and the state machine.
- Callbacks may fire outside the main task; schedule application mutations with `Application::Schedule()` or event bits.
- Never block the main event loop or audio tasks. Avoid unbounded queues in audio paths.

## Build, Test, and Development Commands

Source the ESP-IDF environment first:

```sh
source /path/to/esp-idf/export.sh
idf.py --version
```

**Discover and build:**

```sh
# List all known board/variant names
python3 scripts/release.py --list-boards

# Canonical variant build (changes local sdkconfig)
python3 scripts/release.py <board-directory> --name <variant-name>

# Host-side tests
python3 -m unittest discover -s scripts/tests -v
```

## Coding Style & Naming Conventions

- Use the repository `.clang-format` for C/C++ files; only format files you touch.
- File naming: `snake_case.cc` / `snake_case.h` for source; board implementations are named after the board directory.
- Class naming: `PascalCase` (e.g., `Application`, `MqttProtocol`).
- Constants: `kPascalCase` for enums (e.g., `kDeviceStateIdle`), `UPPER_SNAKE_CASE` for preprocessor defines.
- Each board must export exactly one factory via `DECLARE_BOARD(...)`.
- Keep shared message semantics in `Protocol`; verify both transports (WebSocket + MQTT/UDP) when changing its contract.

## Board & Hardware Configuration

Board selection is a coupled chain:

```
config.json -> scripts/release.py -> main/Kconfig.projbuild -> main/CMakeLists.txt -> board source + config.h
```

When adding a board or variant, update every link in the chain. Include unique board identity, correct `idf_target`, flash/partition settings, exactly one `DECLARE_BOARD`, and board documentation (`README.md`). Treat camera, backlight, display, LED, and battery as optional capabilities.

## Testing Guidelines

- **Board-only change**: build all variants of the affected board; smoke-test changed hardware.
- **Core / audio / protocol / display change**: run host tests and build representative chip/network paths.
- **Protocol changes**: verify both WebSocket and MQTT/UDP transports.
- **Audio changes**: verify capture, playback, wake/VAD, interruption, and reconnect paths.
- A successful build is not hardware validation; report what was tested and what needs physical hardware.

## Commit & Pull Request Guidelines

- Git history follows Conventional Commits style (`feat:`, `fix:`, `refactor:`, `docs:`, `chore:`).
- PRs should include a description of the change, affected boards/variants, and build verification results.
- Never alter an existing boards pins to support different hardware; add a uniquely named board or release variant.
- Do not manually edit generated/vendor output: `build/`, `releases/`, `managed_components/`, `sdkconfig*`, or generated locale/asset headers.
