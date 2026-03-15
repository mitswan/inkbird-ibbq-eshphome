# Inkbird iBBQ ESPHome Integration

Pure ESPHome YAML config for Inkbird iBBQ Bluetooth thermometers (tested with IBT-6XS and IBT-2X). No custom components needed.

**[Jump straight to the code](ibbq-esphome.yaml)**

Uses ESPHome's built-in `ble_client` to connect via GATT, authenticate, and subscribe to realtime temperature notifications. Supports multiple devices simultaneously.

## How it works

The iBBQ protocol requires three writes after connecting:

1. **Pairing key** to `FFF2` (AccountAndVerify)
2. **Enable realtime data** to `FFF5` (SettingsData)
3. **Set units** to `FFF5` (SettingsData)

Temperature data then streams via notifications on `FFF4` (RealtimeData) at ~1Hz. Each probe is 2 bytes, little-endian, in units of 0.1°C.


## Usage

1. Copy the relevant sections from `ibbq-esphome.yaml` into your ESPHome config
2. Replace the MAC addresses with your own (use the `ble_scanner` text sensor to find them)
3. Adjust the number of probe sensors to match your device (2 for IBT-2X, 4 for IBT-4XS, 6 for IBT-6XS, etc.)
4. If you only have one device, remove the second `ble_client` and its sensors

### Connecting and authenticating

On connect, we wait 2s for the BLE stack to be fully ready, then send the three protocol commands in sequence. The delays between writes give the device time to process each command. The same sequence works for all iBBQ models -- only the `mac_address` and `id` change:

```yaml
ble_client:
  # --- IBT-6XS (6 probes) ---
  - mac_address: "YOUR_IBT6XS_MAC_ADDRESS"
    id: ibt6xs_client
    on_connect:
      then:
        - delay: 2s
        # Pairing key (15 bytes) to FFF2 (AccountAndVerify)
        - ble_client.ble_write:
            id: ibt6xs_client
            service_uuid: "FFF0"
            characteristic_uuid: "FFF2"
            value: [0x21, 0x07, 0x06, 0x05, 0x04, 0x03, 0x02, 0x01, 0xb8, 0x22, 0x00, 0x00, 0x00, 0x00, 0x00]
        - delay: 500ms
        # Enable realtime data on FFF5 (SettingsData)
        - ble_client.ble_write:
            id: ibt6xs_client
            service_uuid: "FFF0"
            characteristic_uuid: "FFF5"
            value: [0x0B, 0x01, 0x00, 0x00, 0x00, 0x00]
        - delay: 500ms
        # Set units to Celsius on FFF5 (SettingsData)
        - ble_client.ble_write:
            id: ibt6xs_client
            service_uuid: "FFF0"
            characteristic_uuid: "FFF5"
            value: [0x02, 0x00, 0x00, 0x00, 0x00, 0x00]

  # --- IBT-2X (2 probes) - same protocol, different MAC ---
  - mac_address: "YOUR_IBT2X_MAC_ADDRESS"
    id: ibt2x_client
    on_connect:
      # ... identical sequence with id: ibt2x_client
```

### Handling disconnects

When the iBBQ disconnects (e.g. powered off, out of range), all probes are set to NAN so Home Assistant shows them as unavailable. ESPHome's `ble_client` will automatically attempt to reconnect:

```yaml
    on_disconnect:
      then:
        - lambda: |-
            ESP_LOGD("ibbq", "IBT-6XS Disconnected!");
            id(ibt6xs_probe_1).publish_state(NAN);
            id(ibt6xs_probe_2).publish_state(NAN);
            # ... etc for each probe
```

### Receiving temperature data

Each device gets its own `ble_client` sensor that subscribes to notifications on `FFF4`. `update_interval: never` is important -- the iBBQ only supports notify, not read, so polling would cause errors.

The lambda parses the notification payload: each probe is 2 bytes little-endian in 0.1°C units. Values >= 60000 (typically `0xFFF6`) mean no probe is connected and are published as NAN. The number of probes is determined dynamically from the payload size, so the same lambda works for 2, 4, and 6 probe models:

```yaml
sensor:
  # --- IBT-6XS data listener ---
  - platform: ble_client
    type: characteristic
    ble_client_id: ibt6xs_client
    service_uuid: "FFF0"
    characteristic_uuid: "FFF4"
    notify: true
    update_interval: never
    id: ibt6xs_data_listener
    internal: true
    lambda: |-
      if (x.size() < 2) return 0;
      int num_probes = x.size() / 2;
      if (num_probes > 6) num_probes = 6;
      sensor::Sensor *sensors[] = {
        id(ibt6xs_probe_1), id(ibt6xs_probe_2), id(ibt6xs_probe_3),
        id(ibt6xs_probe_4), id(ibt6xs_probe_5), id(ibt6xs_probe_6)
      };
      int probes_connected = 0;
      for (int i = 0; i < num_probes; i++) {
        uint16_t raw = x[i*2] | (x[i*2+1] << 8);
        if (raw > 0 && raw < 60000) {
          sensors[i]->publish_state(raw / 10.0);
          probes_connected++;
        } else {
          sensors[i]->publish_state(NAN);
        }
      }
      for (int i = num_probes; i < 6; i++) {
        sensors[i]->publish_state(NAN);
      }
      return probes_connected;

  # --- IBT-2X data listener (same pattern, fewer probes) ---
  - platform: ble_client
    type: characteristic
    ble_client_id: ibt2x_client
    # ... same structure, cap num_probes at 2
```

### Probe template sensors

One template sensor per probe exposes the temperature to Home Assistant. Create as many as your device has probes:

```yaml
  # IBT-6XS probes
  - platform: template
    name: "IBT-6XS Probe 1"
    id: ibt6xs_probe_1
    unit_of_measurement: "°C"
    device_class: temperature
    state_class: measurement
    accuracy_decimals: 1
  # ... repeat for probes 2-6

  # IBT-2X probes
  - platform: template
    name: "IBT-2X Probe 1"
    id: ibt2x_probe_1
    unit_of_measurement: "°C"
    device_class: temperature
    state_class: measurement
    accuracy_decimals: 1
  # ... repeat for probe 2
```

## Notes

- Requires `esp-idf` framework (not Arduino) for reliable BLE client support
- The 2s delay after connect avoids a race condition where the write fires before the GATT client is fully ready
- The iBBQ only advertises when not connected to another device -- close the Inkbird phone app before testing
- You can connect multiple iBBQ devices to one ESP32 -- just add more `ble_client` entries and bump `max_connections`

## Acknowledgements

Protocol details from [runningtoy/InkBird_BLE2MQTT](https://github.com/runningtoy/InkBird_BLE2MQTT) -- thank you for working out the iBBQ GATT protocol.
