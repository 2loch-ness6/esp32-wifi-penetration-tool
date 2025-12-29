# ESP32 Wi-Fi Penetration Tool - Copilot Instructions

## Project Overview

This is an **ESP32-based Wi-Fi penetration testing tool** written in **C** using the **ESP-IDF 4.1 framework**. It implements various Wi-Fi attacks including PMKID capture, WPA/WPA2 handshake capture, deauthentication attacks, and denial of service. The tool runs on ESP32 microcontrollers and provides a web-based management interface accessible via a management AP.

**Repository Size**: ~4.5MB (compact codebase)  
**Language**: C  
**Framework**: ESP-IDF 4.1 (commit `5ef1b390026270503634ac3ec9f1ec2e364e23b2`)  
**Target Platform**: ESP32 (ESP32-WROOM-32, ESP32-DEVKITC-32E)  
**Source Files**: ~31 C/H files total (11 in main, 20 in components)

## Build System & Environment Setup

### Prerequisites
**CRITICAL**: This project requires **ESP-IDF 4.1** specifically (commit `5ef1b390026270503634ac3ec9f1ec2e364e23b2`). Newer versions may break compatibility.

**Required Tools**:
- ESP-IDF 4.1 toolchain (xtensa-esp32-elf compiler)
- Python 3.6+ with ESP-IDF requirements installed
- CMake 3.13+ and Ninja 1.8.2+
- `IDF_PATH` environment variable must be set

**Setup Steps** (before any build):
1. Clone ESP-IDF v4.1: `git clone -b v4.1 https://github.com/espressif/esp-idf.git`
2. Checkout specific commit: `cd esp-idf && git checkout 5ef1b390026270503634ac3ec9f1ec2e364e23b2`
3. Install ESP-IDF: `cd esp-idf && ./install.sh esp32`
4. **ALWAYS source environment before building**: `. $IDF_PATH/export.sh` (every new terminal session)
5. Install Python requirements: `python -m pip install -r $IDF_PATH/requirements.txt`

### Build Process

**To build the project**:
```bash
# ALWAYS source ESP-IDF environment first
. $IDF_PATH/export.sh

# Build from project root
cd /path/to/esp32-wifi-penetration-tool
idf.py build
```

**Build output locations**:
- Main binary: `build/esp32-wifi-penetration-tool.bin`
- Bootloader: `build/bootloader/bootloader.bin`
- Partition table: `build/partition_table/partition-table.bin`

**Common Build Issues**:
- **Missing IDF_PATH**: Error "idf.py: command not found" → Run `. $IDF_PATH/export.sh`
- **Wrong ESP-IDF version**: Compile errors → Verify using commit `5ef1b390026270503634ac3ec9f1ec2e364e23b2`
- **Missing submodules**: Build failures → ESP-IDF requires `git submodule update --init --recursive`
- **Component linking errors**: Ensure `CMakeLists.txt` in components properly register dependencies

**Configuration**:
- Default config: `sdkconfig.defaults` (sets `CONFIG_ESP32_WIFI_NVS_ENABLED=n`)
- Customize via: `idf.py menuconfig` (creates `sdkconfig` - gitignored)
- Management AP settings: In menuconfig → "Wi-Fi Controller" → "Management AP"

### Flashing

**With ESP-IDF installed**:
```bash
idf.py flash
```

**Without ESP-IDF** (using pre-built binaries):
```bash
esptool.py -p /dev/ttyS5 -b 115200 --after hard_reset write_flash \
  --flash_mode dio --flash_freq 40m --flash_size detect \
  0x8000 build/partition_table/partition-table.bin \
  0x1000 build/bootloader/bootloader.bin \
  0x10000 build/esp32-wifi-penetration-tool.bin
```

### Testing & Validation

**No automated tests exist** - this is an embedded firmware project without test infrastructure.

**Manual validation**:
1. Flash firmware to ESP32 device
2. Power on ESP32
3. Connect to Management AP (default SSID: `ManagementAP`, password: `mgmtadmin`)
4. Navigate to `192.168.4.1` in browser
5. Verify web UI loads and displays configuration interface

**Validation after code changes**:
- Build succeeds without errors
- Web UI remains functional after HTML/webserver changes
- Component changes should be verified by building the project

## Project Architecture

### Directory Structure
```
/
├── main/                    # Main component (attack implementations, entry point)
│   ├── main.c              # Entry point: initializes event loop, starts mgmt AP, webserver
│   ├── attack*.c/h         # Attack implementations (handshake, PMKID, DoS, deauth)
│   └── CMakeLists.txt      # Registers main component sources
├── components/             # Reusable components (6 total)
│   ├── wifi_controller/   # Wi-Fi operations wrapper (AP, STA, scanning, sniffer)
│   ├── webserver/         # Web UI and HTTP endpoints
│   ├── frame_analyzer/    # 802.11 frame parsing and filtering
│   ├── pcap_serializer/   # PCAP format serialization for captures
│   ├── hccapx_serializer/ # HCCAPX format for hashcat
│   └── wsl_bypasser/      # Bypasses Wi-Fi stack to send raw 802.11 frames
├── build/                 # Build output (pre-built binaries committed)
├── doc/                   # Documentation and diagrams
│   ├── ATTACKS_THEORY.md  # Theory behind attacks
│   ├── images/            # UI screenshots, hardware photos
│   └── drawio/            # SVG diagrams
├── CMakeLists.txt         # Root CMake project file (boilerplate)
├── sdkconfig.defaults     # Default ESP-IDF configuration
├── Doxyfile              # Doxygen configuration for API docs
└── README.md             # Main documentation
```

### Component Architecture

**main** (pseudo-component):
- Entry point in `main.c`: calls `app_main()`
- Initializes event loop, starts management AP, attack framework, webserver
- Contains attack implementations using other components

**wifi_controller** (3 submodules):
- `wifi_controller.c`: Start/stop AP, STA, set MAC addresses
- `ap_scanner.c`: Scan nearby APs, store results
- `sniffer.c`: Promiscuous mode, raw 802.11 frame capture, filtering
- Events: Publishes `SNIFFER_EVENTS` to event loop
- Configuration: `Kconfig` for management AP settings

**webserver**:
- Built on ESP-IDF's `esp_http_server` component
- HTML pages stored in `pages/` as gzipped C header files
- Utility: `utils/convert_html_to_header_file.sh` converts HTML → header
- Endpoints: `/`, `/status`, `/reset`, `/ap-list`, `/run-attack`, `/capture.pcap`, `/capture.hccapx`
- JavaScript client makes AJAX calls to control attacks

**frame_analyzer**:
- Parses 802.11 frames (data frames, EAPOL, management frames)
- Filtering: Listens for `SNIFFER_EVENTS`, matches criteria, emits `DATA_FRAME_EVENTS`
- Provides frame structure definitions in `interface/frame_analyzer_types.h`

**pcap_serializer**:
- Formats frames to LibPCAP format for Wireshark compatibility
- API: `pcap_serializer_init()`, `pcap_serializer_append_frame()`, `pcap_serializer_get_buffer()`

**hccapx_serializer**:
- Formats WPA handshakes to hashcat HCCAPX format
- API: `hccapx_serializer_init()`, `hccapx_serializer_add_frame()`, `hccapx_serializer_get()`

**wsl_bypasser**:
- **CRITICAL**: Uses linker flag `-Wl,-zmuldefs` in `CMakeLists.txt` to override `ieee80211_raw_frame_sanity_check()`
- Allows sending deauthentication and other blocked 802.11 frames
- Overrides ESP-IDF Wi-Fi stack function - do not modify this component unless necessary

### CMakeLists.txt Pattern

Each component has a `CMakeLists.txt` following this pattern:
```cmake
idf_component_register(
    SRCS "file1.c" "file2.c"
    INCLUDE_DIRS "interface"
)
```

The `wsl_bypasser` component additionally uses:
```cmake
target_link_libraries(${COMPONENT_LIB} -Wl,-zmuldefs)
```

## Development Workflow

### Making Code Changes

1. **Always source ESP-IDF environment** before any build command
2. Modify source files in `main/` or `components/`
3. If changing component interface headers in `components/*/interface/`, ensure dependent components are updated
4. Build with `idf.py build` to verify compilation
5. Flash to device for functional testing (build verification alone is insufficient)

### Modifying Web UI

1. Edit HTML in `components/webserver/utils/index.html`
2. Run conversion script: `cd components/webserver/utils && ./convert_html_to_header_file.sh index.html`
3. Script generates gzipped header file in `components/webserver/pages/page_index.h`
4. Rebuild project with `idf.py build`

### Documentation

- **Code documentation**: Uses Doxygen notation (Javadoc-style `/** */` comments)
- **Generate API docs**: Run `doxygen` from repository root → outputs to `doc/api/html/` (gitignored)
- **Component docs**: Each component has a `README.md` explaining its purpose and API

### Configuration Options

- Management AP SSID/password: `idf.py menuconfig` → "Wi-Fi Controller" → "Management AP"
- Max scanned APs: Configurable via Kconfig (default: 20)
- Max AP connections: Configurable via Kconfig (default: 1)

## Key Files Reference

**Root directory**:
- `CMakeLists.txt`: Minimal boilerplate, includes ESP-IDF project setup
- `sdkconfig.defaults`: Sets `CONFIG_ESP32_WIFI_NVS_ENABLED=n` (disables NVS for Wi-Fi)
- `Doxyfile`: Doxygen configuration (PROJECT_NAME: "ESP32 Wi-Fi Penetration Tool")
- `.gitignore`: Excludes build artifacts except specific binaries, sdkconfig, .vscode

**Main sources**:
- `main/main.c`: 32-line entry point, calls setup functions
- `main/attack.c/h`: Attack framework and initialization
- `main/attack_handshake.c/h`: WPA handshake capture logic
- `main/attack_pmkid.c/h`: PMKID capture
- `main/attack_dos.c/h`: Denial of service implementation
- `main/attack_method.c/h`: Deauthentication methods (broadcast, rogue AP)

## Important Notes

- **ESP-IDF version is critical**: Do not upgrade ESP-IDF version without thorough testing
- **Pre-built binaries**: Located in `build/` and committed to repo for users without ESP-IDF setup
- **No legacy make support**: Only CMake/idf.py build system is supported
- **Educational purpose**: Tool is for authorized penetration testing only
- **Hardware target**: Designed for ESP32-WROOM-32 modules (ESP32-DEVKITC-32E dev boards)

## Instructions for Agents

**Trust these instructions first** - only search the codebase when information here is incomplete or appears incorrect. This document is comprehensive and validated. Use grep/glob for specific code searches only when necessary to locate exact function definitions or verify implementation details not covered here.
