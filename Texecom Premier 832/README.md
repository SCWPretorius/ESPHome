# ESPHome Texecom Premier Simple Protocol Monitor

This ESPHome project allows you to monitor the status of Texecom Premier series alarm panels (specifically tested based on the protocol for Premier 832 v16.0) using their "Simple Protocol" over a serial connection. It integrates with Home Assistant to provide entities for zone status and partition armed status.

**This configuration provides MONITORING capabilities only. It cannot be used to arm, disarm, or otherwise control the alarm panel.**

## Features

*   Connects to the Texecom panel via UART (Serial).
*   Authenticates using the panel's UDL password (stored securely in `secrets.yaml`).
*   Polls the panel periodically (approx. every 5-10 seconds) for status updates.
*   Reports the status (Triggered/Healthy) for each configured zone as individual binary sensors in Home Assistant.
*   Reports the status (Armed/Disarmed) for up to 4 partitions as individual text sensors in Home Assistant.
*   Provides customizable names and device classes for zones for better representation in Home Assistant (e.g., Door, Window, Motion).
*   Reports the overall connection/authentication status.
*   Includes fallback WiFi AP and OTA update capabilities.

## Hardware Requirements

1.  **ESP32 or ESP8266 Board:** An ESP32 is generally recommended for better performance.
2.  **Level Shifter:** Texecom panels typically use serial levels different from the ESP's 3.3V TTL logic. A level shifter is **required** to safely connect the ESP's UART pins to the panel's COM port. **Do NOT connect ESP pins directly to the panel's COM port.**
3.  **Wiring:** Jumper wires (Dupont cables) for connections.
4.  **Power Supply:** A suitable power supply for your ESP board.

## Panel Configuration (Required!)

You **must** configure the Texecom Premier panel correctly using the Installer/Engineer menu:

1.  **Enable COM Port:** Select a COM port on the panel to use for this connection and ensure it is **Enabled**.
2.  **Set Protocol/Mode:** Configure the selected COM port's mode to **"PC Com"** or the equivalent setting for your firmware version that matches the documented protocol.
3.  **Set Serial Parameters:** Configure the port settings to **exactly 19200 baud, 8 data bits, None parity, 2 stop bits (19200 8N2)**.
4.  **UDL Password:** You need to know the panel's **UDL Password** (minimum 4, maximum 8 characters). This password will be stored in your `secrets.yaml` file.

## ESPHome Configuration Setup

1.  **Copy YAML:** Save the provided ESPHome YAML configuration (e.g., as `texecom-monitor.yaml`).
2.  **Secrets (`secrets.yaml`):** Create or edit a `secrets.yaml` file in your ESPHome configuration directory. Add entries for your WiFi credentials and your Texecom UDL password:
    ```yaml
    # secrets.yaml
    wifi_ssid: "YourNetworkSSID"
    wifi_password: "YourWiFiPassword"

    # Texecom UDL Password (Min 4, Max 8 chars)
    # IMPORTANT: Keep the single quotes AND the inner double quotes!
    texecom_udl_password: '"YOUR_UDL_PASSWORD_HERE"'
    ```
3.  **Mandatory Customization (Main YAML):** Edit your main YAML file (`texecom-monitor.yaml`):
    *   **`globals:`**
        *   `g_number_of_zones`: Set the total number of zones configured on your panel (e.g., `8`, `16`, `24`, `32`).
        *   *(Note: `g_udl_password` now uses `!secret texecom_udl_password` and reads from `secrets.yaml`)*
    *   **`uart:`**
        *   `tx_pin`, `rx_pin`: Set these to the actual GPIO pins on your ESP board connected to the level shifter's TTL side.
    *   **`esp32:` / `esp8266:`**
        *   `board`: Specify the correct board type you are using (e.g., `nodemcu-32s`, `d1_mini`).
4.  **Optional Customization (Main YAML):**
    *   **`binary_sensor:` (Zones):**
        *   Change the `name:` for each `zone_X_triggered` sensor to be descriptive (e.g., "Living Room PIR", "Front Door Contact").
        *   Change the `device_class:` for each zone sensor to best match its type (e.g., `door`, `window`, `motion`, `smoke`, `tamper`, `problem`). This improves the display in Home Assistant.
        *   For unused zones, you can set `internal: true` to hide them from Home Assistant by default.
    *   **`text_sensor:` (Partitions):**
        *   Change the `name:` for each `partition_X_status_text` sensor (e.g., "House Status", "Garage Status").
    *   **`ota:`:** Set a secure OTA password.
    *   **`wifi: ap:`:** Set a secure password for the fallback Access Point.
    *   **`logger:`:**
        *   Adjust the `level` (e.g., `DEBUG`, `VERBOSE`, `INFO`).
        *   **IMPORTANT:** If your ESP board uses the UART pins for logging (`GPIO1`, `GPIO3`) and these conflict with your chosen `tx_pin`/`rx_pin`, you **must** disable hardware UART logging by setting `baud_rate: 0`. Logs can still be viewed via WiFi/API.
5.  **Compile & Upload:** Use the ESPHome dashboard or command line to compile and upload the configuration to your ESP board.

## Home Assistant Integration

Once the ESP connects to Home Assistant via the API, the following entities will be created:

*   **Zone Status:** `binary_sensor.zone_X_triggered` (e.g., `binary_sensor.front_door`) for each configured zone. The state display depends on the `device_class` (e.g., Open/Closed, Detected/Clear). Unused zones marked `internal: true` will be hidden by default.
*   **Partition Status:** `text_sensor.partition_X_status_text` (e.g., `sensor.partition_1_status`) for partitions 1-4. Displays "Armed" or "Disarmed".
*   **Connection Status:** `text_sensor.panel_status`. Displays the connection and authentication state.

You can add these entities to your Lovelace dashboards and use them in automations. Note that the underlying `binary_sensor.partition_X_armed` entities are marked as `internal: true` but can still be used in Home Assistant automations if needed (they represent ON/OFF).

## Troubleshooting

*   **Check ESPHome Logs:** Set the `logger:` level to `DEBUG` or `VERBOSE` for detailed information. Use the "Logs" option in the ESPHome dashboard.
*   **"Disconnected" or "Timeout?" Status:**
    *   **Verify Wiring:** Check GND (ESP <-> Level Shifter <-> Panel), TX/RX connections (crossed correctly?), physical pin connections on ESP.
    *   **Verify Level Shifter:** Check power and wiring according to its specific type.
    *   **Verify Panel Configuration:** Is the COM port **Enabled**? Is the **Mode** set to Simple/ASCII/Crestron? Are the serial settings **19200 8N2**?
*   **"Authenticating..." Status (Stuck):**
    *   If logs show **no `[parser_raw_rx]` bytes**, it's a hardware/panel config issue (see above).
    *   If logs show **`[parser_raw_rx]` bytes but not `OK` or `ERR0R`**, the panel is likely in the wrong COM port mode.
    *   If logs show **`[parser]` checking for OK/ERR0R but "NO MATCH"**, verify the received bytes against expected (`OK` = `4F 4B 0D 0A`, `ERR0R` = `45 52 52 30 52 0D 0A`).
*   **Panel Returns `ERR0R`:** The UDL password (`texecom_udl_password`) in your `secrets.yaml` file is likely incorrect (ensure it includes the inner `"` quotes), or the panel configuration prevents login via the simple protocol.
*   **Incorrect Zone/Partition States:** Check the `[zone_update]` and `[partition_update]` logs (DEBUG level) to see the raw bytes received and the state being published. Verify this against the actual panel status. The interpretation of partition flags might vary slightly between firmware versions.
*   **Loopback Test:** To test the ESP UART pins/config, disconnect the panel/level shifter and connect a jumper wire directly between the ESP's configured `tx_pin` and `rx_pin`. You should see the sent command bytes echoed back in the `[parser_raw_rx]` logs.

## Limitations

*   **Monitoring Only:** This configuration cannot control the alarm panel (arm, disarm, bypass).
*   **Polling Based:** Status updates rely on polling the panel every ~5-10 seconds, so there will be a slight delay compared to instant events.
*   **Simple Protocol:** This uses the limited "Simple Protocol". More advanced features require different protocols and potentially different hardware interfaces (like the Texecom serial COM-IP module or other integrations).
*   **Partition Status Interpretation:** The armed/disarmed status relies on interpreting bits in specific panel flags (16 and 17). While this works based on the provided documentation, minor variations might exist on different firmware.

## License

This project is licensed under the MIT License.

## Acknowledgements

@RoganDawes [Wintex protocol](https://github.com/RoganDawes/ESPHome_Wintex/tree/main)

@girlpunk [Python Utilities for monitoring Texecom alarm systems](https://github.com/girlpunk/texecom-python/tree/master)

@Samyoue [TexecomVeraPlugin](https://github.com/ludeeus/integration_blueprint)

@shuckc [pialarm](https://github.com/shuckc/pialarm)
