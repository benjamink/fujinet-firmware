# FujiNet Firmware — AI Coding Instructions

**Big Picture**
- **Architecture:** Devices inherit `virtualDevice` and use a protocol factory. Buses (SIO/IWM/DriveWire/IEC/AdamNet) own device registration and IO.
- **Protocols:** `NetworkProtocol` implementations (HTTP/FTP/TNFS/SMB) are instantiated by `ProtocolParser` based on URL scheme; parsed via `PeoplesUrlParser`.
- **Buffers:** Network devices share fixed buffers (input/output 65535, special 256). Status formats vary by bus but logic is consistent.

**Build & Run**
- **Use `build.sh`:** Generates PlatformIO config by merging common + board + local overrides; avoid raw `pio`/`cmake`.
- **Setup board:** `./build.sh -s fujinet-atari-v1` → creates [platformio.local.ini](platformio.local.ini).
- **Build loop:** `./build.sh -cbum` (clean, build, upload, monitor). Build all: `./build.sh -a`. Release zip: `./build.sh -z`.
- **Key files:** [build-platforms/](build-platforms/) templates; local overrides in [platformio.local.ini](platformio.local.ini) (`+=` supported); full docs in [build-sh.md](build-sh.md).

**Patterns & Entry Points**
- **Fuji device (Atari):** Control surface, mounting, WiFi in [lib/device/sio/sioFuji.h](lib/device/sio/sioFuji.h).
- **Network devices:** Repeated pattern: `deviceSpec` → parse → instantiate protocol → operate. See [lib/device/sio/network.h](lib/device/sio/network.h), [lib/device/iwm/network.h](lib/device/iwm/network.h), [lib/device/drivewire/network.h](lib/device/drivewire/network.h).
- **Device base/types:** See [lib/device/device.h](lib/device/device.h) and bus-specific device IDs in [lib/bus](lib/bus).

**Bus Details**
- **SIO (Atari):** [lib/bus/sio](lib/bus/sio). Frame-based IO (ACK/ERROR/data). NetSIO adds buffered ops and speed change. High-speed via `FN_HISPEED_INDEX` in PlatformIO flags.
- **IWM (Apple):** [lib/bus/iwm](lib/bus/iwm). SmartPort commands (read/write/open/close). Multiple network units via `network_data_map[unit]`.
- **DriveWire (CoCo):** [lib/bus/drivewire](lib/bus/drivewire). Packet ops (`R/W/G/S/I/T` + NET/CPM). Devices mapped per `device_id` in `_netDev`.

**Configuration & State**
- **fnconfig.ini:** Persistent config stored in SPI Flash and SD. Sections include General, WiFi, Network, Host(1–8), Mount(1–8), Modem, Cassette, CPM, ENABLE. Device slots map to `[Mount1-8]`.
- **GPIO/IDs/errors:** Pins in [lib/hardware](lib/hardware); device IDs under [lib/bus/*/fujiDeviceID.h](lib/bus); errors in [lib/status_error_codes.h](lib/status_error_codes.h).

**Debugging**
- **Verbose logs:** Enable `Debug_printf()` via PlatformIO flags in [platformio.local.ini](platformio.local.ini): `CORE_DEBUG_LEVEL`, `FNCONFIG_DEBUG`, `VERBOSE_PROTOCOL`, `VERBOSE_HTTP`.
- **Code standards:** Install pre-commit checks: `./coding-standard.py --addhook` (tabs/whitespace/clang-format only on changed files).

**FujiNet‑PC (Desktop Emulation)**
- **Target selection:** `FUJINET_TARGET=ATARI|APPLE` (see [CMakeLists.txt](CMakeLists.txt)).
- **Build/run:** `./build.sh -p -b` or `cmake .. -DFUJINET_TARGET=ATARI && cmake --build . --target=dist && ./dist/run-fujinet`.
- **Components:** ESP‑specific in [components](components); desktop variants in [components_pc](components_pc).

**Making Changes**
- **New protocol:** Implement `NetworkProtocol`, register in `ProtocolParser::createProtocol()`, mirror existing handlers in [lib/network-protocol](lib/network-protocol).
- **Cross‑bus consistency:** If changing network device behavior, update SIO/IWM/DriveWire variants and verify with `./build.sh -a`.
- **Feature gates:** Use `#ifdef BUILD_*` and keep error codes consistent with [lib/status_error_codes.h](lib/status_error_codes.h).
