# Inkbird iBBQ ESPHome Integration

Pure ESPHome YAML config for Inkbird iBBQ Bluetooth thermometers (tested with IBT-6XS). No custom components needed.

Uses ESPHome's built-in `ble_client` to connect via GATT, authenticate, and subscribe to realtime temperature notifications.

## How it works

The iBBQ protocol requires three writes after connecting:

1. **Pairing key** to `FFF2` (AccountAndVerify)
2. **Enable realtime data** to `FFF5` (SettingsData)
3. **Set units** to `FFF5` (SettingsData)

Temperature data then streams via notifications on `FFF4` (RealtimeData) at ~1Hz. Each probe is 2 bytes, little-endian, in units of 0.1°C.

A common mistake is writing the settings commands to `FFF2` instead of `FFF5` -- this will cause the device to disconnect after ~30 seconds.

## Usage

1. Copy the relevant sections from `ibbq-esphome.yaml` into your ESPHome config
2. Replace `YOUR_IBBQ_MAC_ADDRESS` with your device's MAC address
3. Adjust the number of probe sensors to match your device (4 for IBT-4XS, 6 for IBT-6XS, etc.)

The key snippets:

```yaml
ble_client:
  - mac_address: "YOUR_IBBQ_MAC_ADDRESS"
    id: ibbq_client
    on_connect:
      then:
        - delay: 2s
        # Pairing key to FFF2
        - ble_client.ble_write:
            id: ibbq_client
            service_uuid: "FFF0"
            characteristic_uuid: "FFF2"
            value: [0x21, 0x07, 0x06, 0x05, 0x04, 0x03, 0x02, 0x01, 0xb8, 0x22, 0x00, 0x00, 0x00, 0x00, 0x00]
        - delay: 500ms
        # Enable realtime data on FFF5
        - ble_client.ble_write:
            id: ibbq_client
            service_uuid: "FFF0"
            characteristic_uuid: "FFF5"
            value: [0x0B, 0x01, 0x00, 0x00, 0x00, 0x00]
        - delay: 500ms
        # Set units to Celsius on FFF5
        - ble_client.ble_write:
            id: ibbq_client
            service_uuid: "FFF0"
            characteristic_uuid: "FFF5"
            value: [0x02, 0x00, 0x00, 0x00, 0x00, 0x00]
```

```yaml
sensor:
  - platform: ble_client
    type: characteristic
    ble_client_id: ibbq_client
    service_uuid: "FFF0"
    characteristic_uuid: "FFF4"
    notify: true
    update_interval: never
    id: ibbq_data_listener
    internal: true
    lambda: |-
      if (x.size() < 2) return 0;
      int num_probes = x.size() / 2;
      if (num_probes > 6) num_probes = 6;
      sensor::Sensor *sensors[] = {
        id(probe_1), id(probe_2), id(probe_3),
        id(probe_4), id(probe_5), id(probe_6)
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
```

## Notes

- Requires `esp-idf` framework (not Arduino) for reliable BLE client support
- If using `bluetooth_proxy`, set `esp32_ble: max_connections: 4` or higher
- The 2s delay after connect is needed to avoid a race condition with service discovery
- Disconnected probes report values >= 60000 (typically `0xFFF6`), which are filtered to NAN

## Acknowledgements

Protocol details from [runningtoy/InkBird_BLE2MQTT](https://github.com/runningtoy/InkBird_BLE2MQTT) -- thank you for working out the iBBQ GATT protocol.
