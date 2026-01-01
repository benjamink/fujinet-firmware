# FujiNet Firmware - AI Coding Instructions

## Project Overview

FujiNet is a hardware device for retro 8-bit computer systems enabling them to participate in the modern Internet. It loads disk images from remote resources via WiFi, replaces physical disk drives, printers, and modems, and offloads complex modern protocols (HTTPS, JSON parsing) that would be impractical on vintage hardware. It can also work locally via microSD card without networking.

**Platform Support** (from most to least mature):
- **Atari 8-bit** (SIO/0x0-0x7x): Complete/Stable - D:, P:, R:, N: devices, CP/M support
- **Coleco ADAM** (AdamNet): Complete/Stable
- **Apple II/III** (SmartPort & Disk II): Near Complete - disk and network support
- **Commodore 64** (IEC): Mostly Working - FujiNet + Meatloaf
- **CoCo** (DriveWire): Working - disk and network support
- **Atari Lynx**, **RC2014**, **H89**, **Mac68k**, **S100**: In Progress/Prototype
- **ZX Spectrum**: Design phase

**Build Architectures:**
- **ESP-IDF**: Primary build via PlatformIO for ESP32 hardware (ESP, idf_component.yml, CMakeLists.txt)
- **FujiNet-PC**: Linux/macOS/Windows desktop emulation via CMake (CMAKE_BUILD_TYPE=Debug/Release, FUJINET_TARGET=ATARI/APPLE)
- **Multi-target support**: Defined by `BUILD_*` preprocessor flags (BUILD_ATARI, BUILD_APPLE, BUILD_COCO, BUILD_IEC, BUILD_ADAM, BUILD_LYNX, BUILD_RC2014, BUILD_H89, BUILD_S100)

## Critical Build & Development Workflows

### Building Firmware
Use `build.sh` (not direct `pio` or `cmake`). It generates platformio configuration by merging:
```bash
# Common → Target-specific → Local overrides
platformio-ini-files/platformio.common.ini
build-platforms/platformio-{BUILD_BOARD}.ini
platformio.local.ini
```

**Essential commands:**
```bash
./build.sh -s fujinet-atari-v1                    # Setup board (creates platformio.local.ini)
./build.sh -b                                     # Build
./build.sh -cbum                                  # Clean, build, upload, monitor
./build.sh -a                                     # Build all platforms (validate changes)
./build.sh -z                                     # Release zip (pulls platformio.zip-options.ini)
./build.sh -cbum -l platformio.local-{name}.ini  # Build specific board variant
```

**Key files:**
- [platformio.local.ini](platformio.local.ini) - Your board definition (git-ignored, uses `+=` to extend `build_flags`)
- [build-platforms/](build-platforms/) - Board templates defining flash layout, partitions, build options
- [.clang-format](.clang-format) - Enforced via coding-standard.py pre-commit hook

### Testing & Code Standards
```bash
./coding-standard.py --addhook   # Install pre-commit validation (tabs, whitespace, clang-format)
```
Only **changed files** are checked, allowing incremental cleanup in legacy code.

## Device Architecture Pattern

All devices inherit from `virtualDevice` and follow the **protocol factory pattern**. Device types are registered per-bus (SIO, IWM, RS232, etc.).

**Fuji Device** (Control/Configuration, Device ID 0x70 on Atari/SIO):
- Central hub for WiFi configuration, network setup, and device mounting
- SIO commands like 0xDC (Open App Key), 0xDA (Get SSID), 0xD8 (Mount), etc.
- Manages device slot state (which disk images are mounted)
- Provides firmware version, network status, and time information
- [lib/device/sio/sioFuji.h/cpp](lib/device/sio/sioFuji.h) - Full control surface for Atari

**Network device pattern (heavily duplicated across targets):**
```cpp
// Example: lib/device/sio/network.h + network.cpp
class sioNetwork : public virtualDevice {
    std::string deviceSpec;                          // URL passed from host (e.g., "N:HTTP://...")
    std::unique_ptr<PeoplesUrlParser> urlParser;    // Parses deviceSpec into URL components
    NetworkProtocol *protocol;                      // Runtime protocol instance (HTTP, FTP, TNFS)
    ProtocolParser *protocolParser;                 // Factory for instantiating protocols
    NetworkStatus status;                           // Response buffer + error codes
    
    void parse_and_instantiate_protocol() {         // Core pattern in ALL network devices
        create_devicespec();                        // Sanitize deviceSpec
        create_url_parser();                        // Parse to PeoplesUrlParser
        if (!urlParser->isValidUrl()) {
            status.error = NETWORK_ERROR_INVALID_DEVICESPEC;
            return;
        }
        if (!instantiate_protocol()) {              // Factory creates HTTP/FTP/TNFS handler
            status.error = NETWORK_ERROR_GENERAL;
            return;
        }
    }
};
```

**Same pattern repeated in:** [sioNetwork](lib/device/sio/network.h), [iwmNetwork](lib/device/iwm/network.h), [rs232Network](lib/device/rs232/network.h), [adamNetwork](lib/device/adamnet/network.h), [drivewireNetwork](lib/device/drivewire/network.h), [rc2014Network](lib/device/rc2014/network.h), [h89Network](lib/device/h89/network.h), [s100spiNetwork](lib/device/s100spi/network.h) — **refactoring opportunities exist**.

**Bus integration:** Each bus has distinct device management:
- **SIO (Atari)**: Devices as static globals (sioNetwork, sioClock, sioCPM, etc.)
- **IWM (Apple)**: Multiple network units via `network_data_map[current_network_unit]`
- **DriveWire (CoCo)**: Map-based device storage `_netDev[device_id]`

## Bus-Specific Communication Protocols

### SIO Protocol (Atari) — [lib/bus/sio/](lib/bus/sio/)

**Frame-based serial communication** over UART (standard 19200 baud, configurable up to ~124k via high-speed index).

**Command Frame Structure:**
```cpp
union cmdFrame_t {
    uint8_t device;     // Device ID (0x3x=disk, 0x4x=printer, 0x5x=modem, 0x6x=network, 0x7x=Fuji)
    uint8_t comnd;      // Command (0x52=read, 0x57=write, 0x53=status)
    uint8_t aux1;       // Auxiliary parameter 1
    uint8_t aux2;       // Auxiliary parameter 2
    uint8_t cksum;      // Checksum calculated via sio_checksum()
};
```

**NetSIO Protocol:** Enhanced variant for high-speed network operations with command buffering (NETSIO_*):
- `NETSIO_DATA_BYTE` (0x01): Single byte transfer
- `NETSIO_DATA_BLOCK` (0x02): Multi-byte block transfer
- `NETSIO_SPEED_CHANGE` (0x80): Dynamic baud rate negotiation
- `NETSIO_MOTOR_ON/OFF` (0x21/0x20): Device activity control
- Control lines: PROCEED, INTERRUPT, COMMAND handled via virtual signals

**Key Configuration:**
- Standard baudrate: 19200 Hz (1789790 NTSC / Atari frequency)
- High-speed index: Configurable `FN_HISPEED_INDEX` in platformio.ini (0-40 maps to specific rates)
- Device IDs: 0x31-0x3F (disk D1-D15), 0x40-0x4F (printer), 0x50 (modem/R:), 0x60-0x6F (network/N:), 0x70 (Fuji), 0xA0 (CPM/Z:)

**Device Response Pattern:**
1. Receive command frame → calculate checksum
2. Send ACK byte ('A', 0x41) if valid
3. If write operation: receive data → STATUS completion byte
4. If read operation: send data → receive checksum from Atari
5. Errors trigger ERROR byte (0x45) with error code in status

### IWM Protocol (Apple II) — [lib/bus/iwm/](lib/bus/iwm/)

**SmartPort block-level protocol** with packet encapsulation (not character-based like SIO).

**SmartPort Command Codes:**
```cpp
enum {
    SP_CMD_STATUS     = 0x00,  // Get device status
    SP_CMD_READBLOCK  = 0x01,  // Read 512-byte block
    SP_CMD_WRITEBLOCK = 0x02,  // Write 512-byte block
    SP_CMD_OPEN       = 0x06,  // Open file/channel
    SP_CMD_CLOSE      = 0x07,  // Close file/channel
    SP_CMD_READ       = 0x08,  // Read from channel
    SP_CMD_WRITE      = 0x09,  // Write to channel
    // Extended versions (0x40+) for newer systems
};
```

**Status Codes (bitfield):**
```cpp
#define STATCODE_BLOCK_DEVICE    0x80  // Block device=1, character=0
#define STATCODE_WRITE_ALLOWED   0x40
#define STATCODE_READ_ALLOWED    0x20
#define STATCODE_DEVICE_ONLINE   0x10  // Disk in drive or device ready
#define STATCODE_FORMAT_ALLOWED  0x08
#define STATCODE_WRITE_PROTECT   0x04
#define STATCODE_DEVICE_OPEN     0x01  // Character devices only
```

**Device Types (for DIB - Device Information Block):**
- `SP_TYPE_BYTE_HARDDISK` (0x02): Hard disk emulation
- `SP_TYPE_BYTE_FUJINET_NETWORK` (0x11): FujiNet network device
- `SP_TYPE_BYTE_FUJINET_CPM` (0x12): CPM support
- `SP_TYPE_BYTE_FUJINET_CLOCK` (0x13): Real-time clock
- `SP_TYPE_BYTE_FUJINET_PRINTER` (0x14): Printer emulation

**Multi-Unit Support:** Apple supports multiple network units (0-3) via `network_data_map[unit_id]`. Each unit maintains independent:
- URL parser and protocol instance
- Receive/transmit/special buffers (65535/65535/256 bytes)
- Connection state and error codes

**Data Transfer:** Uses 512-byte block alignment for disk operations; character devices handle variable-length reads/writes via channels.

### DriveWire Protocol (CoCo) — [lib/bus/drivewire/](lib/bus/drivewire/)

**Packet-based protocol** over serial (57600 baud) with separate opcodes for disk, serial, and special operations.

**Operation Codes:**
```cpp
#define OP_READ         'R' (0x52)   // Read sector
#define OP_WRITE        'W' (0x57)   // Write sector
#define OP_GETSTAT      'G'          // Get status
#define OP_SETSTAT      'S'          // Set status
#define OP_INIT         'I'          // Initialize device
#define OP_TERM         'T'          // Terminate device
#define OP_SERREAD      'C'          // Serial read
#define OP_SERWRITE     0xC3         // Serial write
#define OP_FUJI         0xE2         // FujiNet special
#define OP_NET          0xE3         // Network operations
#define OP_CPM          0xE4         // CPM operations
#define OP_TIME         '#'          // Time/clock operations
```

**Feature Flags (advertised during DWINIT 'Z'):**
```cpp
#define FEATURE_EMCEE    0x01
#define FEATURE_DLOAD    0x02  // Disk load
#define FEATURE_HDBDOS   0x04  // Hard disk support
#define FEATURE_PRINTER  0x10
#define FEATURE_SSH      0x20
#define FEATURE_PLAYSND  0x40
```

**Device Management:** Map-based per device_id (created on-demand):
```cpp
std::map<uint8_t, drivewireNetwork*> _netDev;  // _netDev[device_id] = new drivewireNetwork()
```

**Sector-Based Transfer:** Disk operations work with 256-byte sectors; network device can override with custom packet formats. Includes sophisticated retry logic for noisy serial connections.

## Protocol & Network System

**Protocol hierarchy:**
- `NetworkProtocol` (abstract base in [lib/network-protocol/Protocol.h](lib/network-protocol/Protocol.h)) - Defines interface (open, read, write, special_80, perform_idempotent_80)
- Concrete implementations: HTTP, FTP, TNFS, SMBS (in lib/network-protocol/)
- `ProtocolParser` factory - Creates protocol by URL scheme/method
- `PeoplesUrlParser` - Parses deviceSpec into URL components

**Buffer management:** All network devices use fixed buffers:
- INPUT_BUFFER_SIZE, OUTPUT_BUFFER_SIZE, SPECIAL_BUFFER_SIZE = 65535 / 65535 / 256
- Status byte structure varies by target (SIO vs RS232 vs IWM) but logic is identical

## Code Organization & Patterns

**Build configuration constants are scattered:**
- GPIO pins: [lib/hardware/](lib/hardware/)
- Device IDs: [lib/bus/*/fujiDeviceID.h](lib/bus/)
- Error codes: [lib/status_error_codes.h](lib/status_error_codes.h)
- Network errors: Inside each network.h file

**Macros for target selection:** Files use conditional compilation (`#ifdef BUILD_ATARI`, `#ifdef ESP_PLATFORM`). Check [lib/device/device.h](lib/device/device.h) for device type definitions.

**Debugging:** Extensive use of `Debug_printf()` macro. Enable with `CORE_DEBUG_LEVEL` and target-specific flags in platformio.local.ini:
```ini
[env]
build_flags +=
    -D CORE_DEBUG_LEVEL=5
    -D FNCONFIG_DEBUG=1
    -D VERBOSE_PROTOCOL
    -D VERBOSE_HTTP
```

## Multi-Platform Emulation (FujiNet-PC)

Desktop builds via CMake allow testing without hardware. Key differences:
- **No ESP-IDF headers** - #ifndef ESP_PLATFORM guards used extensively
- **Environment variables control target:** `export FUJINET_TARGET=ATARI` (CMakeLists.txt line 20)
- **Build & run:**
  ```bash
  ./build.sh -p -b    # PlatformIO PC target (if configured)
  # OR
  cd build && cmake .. -DFUJINET_TARGET=ATARI && cmake --build . --target=dist
  cd dist && ./run-fujinet
  ```
- Components duplicated: [components_pc/](components_pc/) (libssh, cJSON, mongoose, etc.) vs [components/](components/) (ESP-specific)

## Configuration Management (fnconfig.ini)

FujiNet reads/writes configuration from [fnconfig.ini](https://github.com/FujiNetWIFI/fujinet-firmware/wiki/FujiNet-Configuration-File%3A-fnconfig.ini), stored in both internal SPI Flash and external SD card.

**Key Sections:**
- **[General]**: Device name, HSIO index (high-speed baud rate), boot mode, timezone, config enable flags
- **[WiFi]**: SSID, passphrase, WiFi enable/disable
- **[Network]**: SNTP server for NTP time sync
- **[Host(1-8)]**: Host definitions (TNFS remote servers or SD card slots) with type (SD/TNFS) and name
- **[Mount1-8]**: Mount configurations linking Host slots to file paths with read/write modes
- **[Modem]**: Modem enable flag, sniffer logging
- **[Cassette]**: Cassette enable, play/record state, pulldown resistor config
- **[CPM]**: CP/M enable flag, CCP file path
- **[ENABLE]**: Per-device-slot enable/disable (8 slots) and APETime clock enable

**Device Slot Mapping (Atari example):**
- Slots 1-8 map to `[Mount1-8]` and control which disk images are mounted where
- APETime (real-time clock) can be enabled independently
- Configuration changes auto-save to both SPI Flash and SD card

## When Modifying This Codebase

**For device/protocol changes:**
1. Check if pattern is repeated across multiple buses (sioNetwork, iwmNetwork, rs232Network, etc.)
2. If yes, consider unified refactor vs. consistent duplication per target
3. Update [lib/network-protocol/Protocol.h](lib/network-protocol/Protocol.h) if protocol interface changes
4. Test ALL builds: `./build.sh -a` validates all platforms

**For new network protocols:**
1. Inherit from `NetworkProtocol`, implement open() / read() / write() / special_* methods
2. Register in `ProtocolParser::createProtocol()` by URL scheme
3. Reuse existing protocol as template (HTTP, FTP, TNFS in [lib/network-protocol/](lib/network-protocol/))

**For platform-specific features:**
- Feature gates: `#ifdef BUILD_*` guards in device files
- Example: IWM (Apple) has multi-unit support; SIO (Atari) uses static globals
- Consistent error codes: reference [lib/status_error_codes.h](lib/status_error_codes.h)

**Code standards non-negotiable:**
- Run `coding-standard.py` before commit (installed via --addhook)
- No trailing whitespace, tabs only in Makefiles, clang-format alignment
- Legacy code isn't cleaned incrementally; new code must comply
