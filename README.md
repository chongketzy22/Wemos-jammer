The uploaded sketch is a large, multifunction ESP32 program designed to turn a Wemos Lolin32 OLED into a versatile radio, IR, and Wi-Fi analysis and control platform with a touchscreen-style OLED interface, buttons for navigation, web-based control, data logging, and multiple modular subsystems (Wi-Fi, nRF24, CC1101, IR, storage). It is organized around a set of device objects, global state, mode/submode abstractions, UI rendering routines for the OLED, network and captive-portal/websocket services for an interactive web UI, and persistent storage for captured data and user settings. The overall design goal appears to be to provide an all-in-one handheld test and development tool for radio/IR/network experimentation, diagnostics, and demonstration. The following is a detailed, comprehensive description covering what the sketch requires, how the hardware should be wired, the libraries it depends on, the major internal functions and their responsibilities, the modes and submodes and what each is intended to do in lawful testing and development contexts, safe use recommendations, and notes on limitations and troubleshooting.

Hardware overview and wiring
This sketch targets the Wemos Lolin32 OLED (ESP32) development board that includes an SSD1306 128×64 I²C OLED. In addition to the onboard display and two user buttons, the sketch integrates support for an nRF24L01 2.4 GHz transceiver, a CC1101 (sub-GHz) transceiver module, IR receive/transmit hardware, and optional peripheral devices/filesystems. The wiring expected by the sketch (typical, but confirm the pin defines at top of the file) is:

* Wemos Lolin32 OLED core:

  * 3.3V: board 3.3 V supply (power modules, transceivers)
  * GND: common ground
  * I²C (OLED): SDA -> GPIO5, SCL -> GPIO4 (the sketch uses Wire.begin(OLED\_SDA, OLED\_SCL) before display.begin()). OLED reset often mapped to GPIO16 or -1; confirm the define. SSD1306 I²C address is 0x3C in the sketch.
  * Buttons: two GPIO inputs used for navigation/action (configured with internal pullups or external resistors; one for short/long press detection). Typical pins in sketches: GPIO34/GPIO35 (input-only pins); adjust wiring to match your board and pin defines.

* nRF24L01 (2.4 GHz radio) typical wiring:

  * VCC -> 3.3 V (do not supply 5 V)
  * GND -> GND
  * CE -> CE pin define in sketch (e.g., GPIO27 or similar)
  * CSN/CS -> CS pin define (often GPIO5 or another)
  * SCK -> GPIO18
  * MOSI -> GPIO23
  * MISO -> GPIO19
  * NOTE: Add a 10 µF to 47 µF decoupling capacitor across VCC and GND pins of the module to prevent brownouts during transmit spikes. Use a stable 3.3 V regulator.

* CC1101 (sub-GHz radio) typical wiring:

  * VCC -> 3.3 V
  * GND -> GND
  * CS -> chip select pin defined in sketch (e.g., GPIO15)
  * SCK -> SPI SCLK (e.g., GPIO14)
  * MOSI -> SPI MOSI (e.g., GPIO13)
  * MISO -> SPI MISO (e.g., GPIO12)
  * GDO0 / GDO2 lines optionally used for interrupts/packet-ready signals (wire to free GPIOs if used)

* IR transmitter and receiver:

  * IR receiver data out -> IR\_RECEIVE\_PIN (e.g., GPIO26 is safe)
  * IR transmitter control -> IR\_SEND\_PIN (an output, good choices are GPIO25 or similar)
  * Transmit/receive modules powered from 3.3 V and GND; ensure correct polarity and check modules that require a transistor/driver for the IR LED to avoid driving it directly from the ESP32 pin.

* Optional: SPIFFS / filesystem uses the same flash on the ESP32; no dedicated wiring. If external sensors/modules are present, ensure they do not conflict with used SPI or I²C pins.

Required software libraries and why they matter
The sketch uses a suite of well-established Arduino/ESP32 libraries to provide device drivers, networking, and storage capabilities. These are required to compile and run the sketch:

* WiFi.h — ESP32 Wi-Fi stack; used for AP mode, station mode, and Wi-Fi scanning.
* esp\_wifi.h — lower-level Wi-Fi functions (promiscuous sniffing, advanced features).
* DNSServer.h — used to implement a captive portal by answering DNS queries while the ESP runs as an AP.
* WebServer.h — small HTTP server to host the web UI and API pages.
* WebSocketsServer.h — bi-directional, real-time communication between web UI and device for telemetry and control.
* SPI.h — hardware SPI for nRF24 and CC1101 modules.
* Wire.h — I²C master controller to communicate with the SSD1306 OLED.
* Adafruit\_GFX.h and Adafruit\_SSD1306.h — high-level graphic routines and SSD1306 driver for the OLED display.
* nRF24L01.h and RF24.h — control and communication with nRF24L01 modules (packet send/receive, channel config).
* ELECHOUSE\_CC1101\_SRC\_DRV.h (or other CC1101 driver) — SPI control of the CC1101 transceiver (frequency, modulation, TX/RX modes).
* IRremoteESP8266.h, IRrecv.h, IRsend.h, IRutils.h — IR receiving, decoding, and sending of IR protocols; useful for learning and replaying remote control codes.
* SPIFFS.h or LittleFS.h — flash filesystem for storing captures, logs, and configuration files.
* EEPROM.h or nvs\_flash.h — nonvolatile storage for small settings and credentials.
* Optional helper libs: RF utilities, JSON or string helpers, timing utilities, and any custom library used by the sketch.

Major software components and architecture
The sketch is modular and follows a structure that separates hardware initialization, UI, mode handling, networking, and persistent storage. Key components:

1. Device objects

   * Instantiate display, radio objects (RF24), SPI class, DNSServer, WebServer, WebSocketsServer, IRrecv/IRsend, and decode\_results. These objects are globally accessible for mode handlers.

2. Forward declarations and function prototypes

   * A large set of prototypes exists at the top of the file, providing the function signatures for menu drawing, mode runners, storage helpers, and ISR callbacks. This lets the compile unit order be flexible.

3. setup()

   * Initializes serial (for debug), Wire.begin(OLED\_SDA, OLED\_SCL), display.begin(), radio.begin(), CC1101 init, IR modules init, SPIFFS.begin(), Wi-Fi/AP startup if a captive portal is used, and registers web endpoints and WebSocket event handlers (webSocket.onEvent). setup() also mounts SPIFFS and optionally prints diagnostics to the OLED.

4. loop()

   * Dresses an event loop that handles user input (handleButtonPress), runs the currently active mode (runActiveModes) while managing timing for display refresh, heartbeat, and periodic tasks. It conditionally handles webServer.handleClient() and dnsServer.processNextRequest() and webSocket.loop() only when the device is in the Wi-Fi Web UI submode, enabling per-mode resource usage and lower power/less interference in other modes.

5. Mode runners

   * For each major mode there is a runXMode() function (runWiFiMode, runNRFMode, runRFMode, runIRMode, runStorageMode). These functions abstract the behavior of that subsystem and call submode handlers for fine control.

6. UI & display

   * updateDisplay(), drawHeader(), drawMenu(), drawActiveScreen(), and a set of drawXScreen() functions render text, progress bars, lists, and signal indicators to the SSD1306. The UI shows mode names, submode selection, RSSI or packet counters, and action prompts. The OLED is updated at a controlled rate (e.g., every 150 ms) to minimize CPU and SPI traffic.

7. Networking & web UI

   * When in Wi-Fi Web UI submode the device runs as an AP and/or connects to a network. The sketch creates an HTTP web UI served from SPIFFS or generated HTML strings, with WebSocket connections for live telemetry. A captive portal via DNSServer redirects clients to the local web UI for configuration and monitoring.

8. Data capture & storage

   * The sketch logs packets, decoded IR codes, and other captured data to SPIFFS (files) and uses a simple ring buffer for recent packet history. It provides functions to list, save, and delete capture files through the storage mode or web UI.

9. Radio & IR utilities

   * The RF and IR subsystems include functions to scan channels, perform passive sniffing (receive and decode packets), and send or replay messages. For legal and ethical reasons the sketch should restrict itself to lawful testing and educational demonstration.

10. Safety & resource management

    * The code contains flags such as actionRunning, nrfConnected, cc1101Connected, spiffs\_mounted to track resource state. The loop avoids starting resource heavy services unless the user explicitly enters a relevant mode, minimizing risk of inadvertent interference and conserving power.

Mode & submode descriptions and lawful use cases
The sketch defines multiple modes, each intended for analysis or benign interaction with wireless devices and IR remotes. Below is a concise but detailed description of each mode and its legitimate, legal uses. Note: the device is capable of analyzing many common wireless signals; it must be used only for lawful testing, diagnostics, security research with owner consent, or development work in controlled environments.

Wi-Fi mode

* Overview: Uses ESP32 built-in Wi-Fi to scan networks, display SSID/BSSID/RSSI, run a captive portal and web UI, and optionally enter a monitor/promiscuous mode to observe management frames for analysis.
* Submodes:

  * Network Scan: Performs active/ passive scans to list SSIDs, channels, and signal strength, useful for site surveys and selecting channels for access point placement.
  * Signal Graph / Heatmap: Displays relative RSSI history for chosen APs for antenna alignment or placement evaluation.
  * Web UI / Captive Portal: Runs an AP and captive portal via DNSServer and WebServer. The web interface allows live telemetry via WebSockets and configuration of device options.
  * Promiscuous Sniffer (Read-only analysis): Collects management/Beacon/Probe header info for analysis of network behavior. Use for network diagnostics and research with permission.
* Lawful use cases:

  * Wi-Fi site surveying, channel planning, debugging rogue APs with permission, and educational demonstration of 802.11 management frames. Do not perform deauthentication or active disruption without explicit authorization.

nRF24 mode

* Overview: Controls an nRF24L01 transceiver to scan 2.4 GHz channels for active communication, decode payloads where possible, perform link tests, and compute packet statistics. Useful for debugging IoT devices and custom radios.
* Submodes:

  * Channel Scanner: Lists channels with observed activity and packet counts; used to detect congestion or interference for your own nRF24 networks.
  * Link Tester: Sends/receives test packets to a paired device (requires another node) to measure latency, packet loss, and RSSI.
  * Passive Sniffer (protocol analysis): For research and reverse engineering of self-owned devices; logs headers and raw payloads for protocol decoding.
* Lawful use cases:

  * Debugging, firmware testing, interoperability checks, and reverse engineering performed only on devices you own or with the owner’s consent. Avoid capturing or injecting into networks you do not have permission to test.

CC1101 / Sub-GHz RF mode

* Overview: Interfaces with a CC1101 sub-GHz transceiver to enumerate frequencies, observe signal presence, and capture packets for analysis. Commonly used in industrial, remote control, or IoT bands (device must respect local frequency allocation and power limits).
* Submodes:

  * Frequency Sweep / Scanner: Measures energy across configured frequency ranges to find activity or interference.
  * Packet Capture (Read-only): Records identifiable packet headers or raw bursts for offline analysis; used to troubleshoot telemetry or link reliability in permitted deployments.
  * Modulation/Parameter Tester: Allows changing data rates and modulation settings to match legitimate endpoints during development.
* Lawful use cases:

  * Testing and development of sub-GHz radio links in lab or with permission; regulatory compliance checks; interference investigation with authorized equipment.

IR mode

* Overview: Uses IRrecv and IRsend to capture infrared remote control signals, decode them into known protocols, and replay stored codes. Includes logging and replay functionality.
* Submodes:

  * IR Capture / Decode: Receives and decodes common IR protocols (NEC, RC5, RC6, etc.), useful for universal remote development and device testing.
  * IR Replay: Replays saved code sequences to control consumer devices for testing automation and integration.
  * IR Code Library: Save commonly used codes in SPIFFS and recall them from the OLED UI or web UI.
* Lawful use cases:

  * Home automation, universal remote testing, research into IR protocols, and integration testing for devices you own.

Storage mode

* Overview: Manages SPIFFS stored captures, log files, and persistent settings. Exposes a filesystem browser on the OLED and web UI with options to download, delete, or replay files.
* Submodes:

  * View Files: List captures and metadata (timestamp, mode, duration).
  * Replay / Load: Load a stored capture to replay in the appropriate transmitter mode (IR replays are benign if used with consent).
  * Erase / Format: Format SPIFFS or delete selected files; useful for cleanup after testing.
* Lawful use cases:

  * Data management, offline analysis, and archiving of legitimate testing sessions.

Key software functions — what each does

* handleButtonPress(): Debounces and classifies short vs long presses; updates navigation state and triggers submode changes or start/stop actions.
* handleNavigation(int clicks, bool isLongPress): Adjusts currentSubMode and selection offset; short presses move the cursor; long presses change submode or toggle advanced menus.
* toggleAction() / stopAllActions(): Begin or end active data capture or transmission sequences; these guard against concurrent conflicting operations.
* runActiveModes(): Dispatcher that calls the appropriate runXMode() based on currentMode and currentSubMode; coordinates hardware state transitions.
* updateDisplay(): Refreshes the OLED; calls helper draw functions to show mode info, packet counters, and telemetry graphs; throttles updates to prevent display flicker.
* drawHeader(), drawMenu(), drawActiveScreen(): Modular drawing routines; header displays uptime and mode; menu shows selectable items; active screen shows live telemetry and controls.
* checkMemory(): Confirms SPIFFS availability and free space; ensures no data loss during writes and avoids running out of memory mid-capture.
* IRAM\_ATTR promiscuousCallback(void\* buf, wifi\_promiscuous\_pkt\_type\_t type): If promiscuous Wi-Fi sniffing is used, this callback is invoked by the Wi-Fi driver to copy packet headers into the sketch for analysis (read-only metadata extraction).
* runWiFiMode(), runNRFMode(), runRFMode(), runIRMode(), runStorageMode(): Implement the sequence of operations for each mode — configure hardware, begin capture or operation, and update UI and storage.
* handlePortalRoot(): Serves the root web UI page; can generate live HTML from templates saved in SPIFFS or produce JSON endpoints for the web UI.
* broadcastSystemStatus(): Periodic function that composes a JSON status packet and transmits it to connected WebSocket clients for live UI updates.
* initJamPatterns(), initJamSchedules(), initProtocolMaps(): Helper initializers for stored patterns and mapping tables — used by signal generation code for benign pattern replay or testing (note: do not use for unlawful interference).

What the sketch can be used for (permissible, ethical, and educational uses)

* Wi-Fi site surveys: measuring RSSI and channel occupancy to choose optimal AP placement and channels for better coverage.
* Protocol development and debugging: testing custom nRF24/CC1101 links, adjusting modulation or parameters for robust comms, validating firmware changes.
* IR remote reverse engineering and automation: capture and rebuild IR macros, create universal remote profiles, automate device testing.
* Device interoperability testing: verify that embedded devices can communicate correctly over nRF24 or sub-GHz links in lab conditions.
* Demonstration and education: teach network concepts, show Wi-Fi management frames, visualize radio spectrum activity (in a legal/test environment).
* Data logging and replay: store captured, self-owned transmissions for later analysis or repeatable tests in controlled setups.
* Web UI remote control: configure and monitor a testbed remotely within the same authorized network.

Limitations, constraints, and safety considerations

* Legal/regulatory compliance: Radio transmissions are regulated. Use this device only in allowed bands and power levels, and only transmit when you own both ends or have explicit permission. Interference, intentional disruption, and jamming are illegal in most jurisdictions. The sketch may include experimental transmit/replay functionality; do not use it to interfere with third-party communications.
* Power & antenna: RF modules can draw short bursts of current during TX; use a stable 3.3 V regulator and decoupling capacitors. Antenna choice and tuning are essential for correct operation and compliance.
* ADC and Wi-Fi concurrency: Some ADC pins (ADC2) are not reliable while Wi-Fi is in use on ESP32. If using analog sensors, prefer ADC1 pins when Wi-Fi is active.
* Boot and strapping pins: Avoid using pins that affect the ESP32 boot mode (e.g., GPIO0, GPIO2, GPIO15) for external modules that could hold them in undesired states during reset.
* SPI and shared bus: When using both nRF24 and CC1101 on SPI, ensure correct CS lines and manage SPI settings (clock polarity and phase) for each device; reconfigure SPI speed when switching devices if required.
* Flash corruption risk: If the ESP32 shows “flash read err” at boot, use the IDE’s “Erase Flash → All Flash Contents” or esptool erase\_flash to fully reset the flash. Ensure reliable USB power during flashing to avoid interrupted writes.
* OLED persistence: The SSD1306 retains its display memory until cleared. Always call display.begin(), display.clearDisplay(), and display.display() during initialization to guarantee a known blank state.

Troubleshooting checklist

* Libraries and board package: Confirm you have the right ESP32 board package installed in Arduino IDE and all listed libraries present and up to date.
* Pin conflicts: Verify no two devices share the same GPIO in a conflicting way (e.g., OLED reset and IR receiver on same pin). Use the sketch’s pin defines to match wiring exactly.
* Power supply: Use a robust 3.3 V supply for RF modules; if you see resets during TX, add a bulk capacitor and ensure regulator can supply peak currents.
* SPIFFS and filesystem: If SPIFFS fails to mount, reformat or erase flash and reupload. Use Tools → Erase Flash → All Flash Contents if necessary.
* Web UI connectivity: If clients can’t reach the captive portal, ensure DNSServer and WebServer are initialized and that the ESP is in AP mode. Also check that webSocket.onEvent is set and that webSocket.loop() is called in the correct mode.
* Debugging output: Use Serial at 115200 for logs. Check for initialization messages from display.begin(), radio.begin(), CC1101 init, and SPIFFS mount status to locate failing subsystems.

Ethical guidance and alternatives to harmful use

* This device is a powerful test and research tool. With power comes responsibility: never use it to intercept, disrupt, or degrade services that you do not own or have explicit authorization to test. Activities like deauthentication, jamming, or unauthorized interception are illegal and harmful.
* If you need to test disruptive behavior for research, perform experiments only in a controlled lab environment with shielded enclosures (Faraday cages), test equipment, and written consent from affected parties, or in designated test frequency bands using proper licensing. Consider using signal generators and dedicated RF test equipment in a certified RF lab.
* For learning radio protocol internals, focus on passive analysis, building simulators, and developing robust, standards-compliant implementations for your own devices rather than interfering with deployed systems.
* If compliance testing or spectrum occupancy studies are needed, partner with accredited testing labs or obtain required permits.

Operational checklist before using the device

1. Confirm wiring matches sketch defines: OLED SDA/SCL, IR pins, nRF24/CC1101 SPI lines and CS pins, button pins.
2. Install required libraries in Arduino IDE / ArduinoDroid and ensure ESP32 board package is current.
3. Upload a simple test sketch to validate OLED and buttons first (display clear + “OK” message).
4. Test each radio module with minimal example sketches (RF24 example, CC1101 example) to confirm module and power integrity.
5. Mount SPIFFS and check free space; create a small test file and read it back.
6. Enter Wi-Fi mode and verify AP/captive portal behavior and WebSocket connectivity from a browser on a phone or laptop.
7. Only after all hardware checks pass, use capture modes to collect data from self-owned devices; store logs to SPIFFS and review on a PC.

Summary of intended capabilities (non-exhaustive)

* Real-time OLED menu UI with multi-level navigation for modes and submodes.
* Wi-Fi scanning, captive portal web UI with live WebSocket telemetry, and read-only promisc sniffing for diagnostics.
* nRF24 and CC1101 support for scanning, link testing, and educational packet capture/analysis.
* IR capture, decode, store, and replay functionality for remote control automation and testing.
* Persistent storage of captures and settings via SPIFFS and NVS/EEPROM.
* Controlled execution model that activates heavy subsystems only in their respective modes to minimize accidental interference and conserve resources.

If you plan to deploy or test this sketch, proceed with care: ensure you adapt pin mappings to your exact Wemos board revision, observe radio regulations in your jurisdiction, power external transceivers properly, and avoid any testing that could affect others without explicit authorization.
