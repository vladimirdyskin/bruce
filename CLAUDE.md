# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Bruce is an ESP32 firmware for offensive security and penetration testing, supporting 28+ hardware variants (M5Stack, Lilygo, CYD, Marauder, custom boards). Written in C++ (Arduino framework) with an embedded JavaScript interpreter (mquickjs). Capabilities include WiFi attacks, BLE operations, RFID/NFC, sub-GHz RF, infrared, GPS, LoRa, NRF24, FM radio, and BadUSB.

## Build System

PlatformIO is the build system. The default target is `m5stack-cplus2`.

```bash
# Build default environment
pio run

# Build specific device
pio run -e m5stack-cardputer
pio run -e lilygo-t-embed-cc1101

# Upload to device
pio run -t upload

# Monitor serial output
pio device monitor

# Build with Docker
docker compose up
# Override target: PIO_ENVS=m5stack-cardputer docker compose up
```

To change the default build target, uncomment the desired environment in `platformio.ini` under `[platformio] default_envs`.

Available environments are listed in `platformio.ini` lines 13-67. LAUNCHER_ prefixed variants are LITE_VERSION builds for M5Launcher compatibility.

## Code Formatting

Uses clang-format (`.clang-format`): LLVM base style, 4-space indentation, no tabs, 110 column limit, attached braces, short statements on single lines allowed.

## Architecture

### Directory Layout

- `src/main.cpp` — Entry point: hardware init, FreeRTOS task creation, main menu loop
- `src/core/` — Core services: display, config, settings, input, SD card, LED, serial CLI
- `src/core/menu_items/` — Main menu categories (WiFi, BLE, RFID, RF, IR, GPS, Scripts, etc.)
- `src/core/serial_commands/` — Serial CLI command handlers
- `src/modules/` — Feature implementations organized by domain:
  - `wifi/` — Deauth, evil portal, sniffer, beacon spam, karma, responder
  - `rfid/` — PN532, MFRC522, Chameleon Ultra, tag emulation, EMV reader
  - `rf/` — CC1101 sub-GHz: scan, send, spectrum, protocols
  - `ble/` — BLE spam, Apple/Samsung attacks, scanning
  - `ir/` — Infrared TX/RX
  - `bjs_interpreter/` — JavaScript engine with hardware API bindings
  - `gps/`, `NRF24/`, `fm/`, `lora/`, `ethernet/`, `pwnagotchi/`, `badusb_ble/`
- `include/` — Global headers: `globals.h`, `precompiler_flags.h`, `MenuItemInterface.h`
- `boards/` — Per-device configs: `<device>/<device>.ini` (build flags, pin defs) and `<device>/interface.cpp` (HAL init)
- `boards/_boards_json/` — PlatformIO board JSON definitions
- `lib/` — Local libraries: HAL, TFT_eSPI, Bad_Usb_Lib, PN532_SRIX, mquickjs headers

### Key Architectural Patterns

**Conditional Compilation**: Features are toggled per-device via `-D` build flags in each board's `.ini` file. Common flags: `HAS_SCREEN`, `LITE_VERSION`, `USE_CC1101_VIA_SPI`, `USE_NRF24_VIA_SPI`, `USB_as_HID`, `HAS_RTC`, `BOARD_HAS_PSRAM`, `MIC_SPM1423`, `FM_SI4713`. Pin assignments (`GROVE_SDA`, `GROVE_SCL`, `CC1101_*`, `NRF24_*`, etc.) are set per-device. Defaults live in `include/precompiler_flags.h`.

**Menu System**: `MainMenu` (in `src/core/main_menu.h`) holds instances of all menu categories. Each menu item extends `MenuItemInterface` (in `include/MenuItemInterface.h`) and implements `optionsMenu()`, `drawIcon()`, `hasTheme()`, and `themePath()`. The options menu populates a global `std::vector<Option>` and calls `loopOptions()` for user selection.

**Global State**: Extensive globals in `include/globals.h` — `bruceConfig`, `bruceConfigPins`, `tft`/`sprite`/`draw` (display), button state (`NextPress`, `SelPress`, `EscPress`, etc.), `wifiConnected`, `sdcardMounted`, `returnToMenu`. Input is checked via the `check()` inline function which suspends/resumes the input handler FreeRTOS task.

**Configuration Persistence**: `BruceConfig` (`src/core/config.h/cpp`) stores settings as JSON in LittleFS. `BruceConfigPins` handles runtime GPIO pin configuration.

**FreeRTOS Tasks**: Input handling runs on a dedicated task (`xHandle`). Serial CLI, JavaScript interpreter, and SSH also use separate tasks with configurable stack sizes (PSRAM-aware defaults in `precompiler_flags.h`).

**Hardware Abstraction**: Each board has `boards/<device>/interface.cpp` implementing device-specific init (power, display, buttons, peripherals). The HAL layer in `lib/HAL/` provides `io_expander` and display abstractions.

### Adding a New Device

1. Create `boards/<device>/` directory with `<device>.ini`, `interface.cpp`, and `pins_arduino.h`
2. Add board JSON to `boards/_boards_json/`
3. Add environment entry to `platformio.ini`
4. Set all pin definitions, feature flags, and TFT_eSPI config in the `.ini` file

### Adding a New Module/Menu Item

1. Create module source in `src/modules/<domain>/`
2. Create menu item header in `src/core/menu_items/` extending `MenuItemInterface`
3. Add the menu to `MainMenu` class in `src/core/main_menu.h`
4. Guard with `#ifdef` if hardware-specific

### Pre/Post Build Scripts

- `patch.py` — Pre-build patching of dependencies
- `pre_build_current_year.py` — Injects build year
- `gen_mqjs_headers.py` — Generates JavaScript interpreter headers (requires 32-bit GCC; path configurable via `mqjs_host_cc` in `platformio.ini`)
- `build.py` — Post-build binary processing
