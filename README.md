# Inkbird iBBQ ESPHome Integration

Pure ESPHome YAML config for Inkbird iBBQ Bluetooth thermometers (tested with IBT-6XS). No custom components needed.

**[Jump straight to the code](ibbq-esphome.yaml)**

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

### BLE connection slots

If you're using `bluetooth_proxy` or other BLE components, make sure you have enough connection slots. The iBBQ client needs one slot for itself:

```yaml
esp32_ble:
  max_connections: 4
```

### Connecting and authenticating

On connect, we wait 2s for the BLE stack to be fully ready, then send the three protocol commands in sequence. The delays between writes give the device time to process each command:

```yaml
ble_client:
  - mac_address: "YOUR_IBBQ_MAC_ADDRESS"
    id: ibbq_client
    on_connect:
      then:
        - delay: 2s
        # Pairing key (15 bytes) to FFF2 (AccountAndVerify)
        - ble_client.ble_write:
            id: ibbq_client
            service_uuid: "FFF0"
            characteristic_uuid: "FFF2"
            value: [0x21, 0x07, 0x06, 0x05, 0x04, 0x03, 0x02, 0x01, 0xb8, 0x22, 0x00, 0x00, 0x00, 0x00, 0x00]
        - delay: 500ms
        # Enable realtime data on FFF5 (SettingsData)
        - ble_client.ble_write:
            id: ibbq_client
            service_uuid: "FFF0"
            characteristic_uuid: "FFF5"
            value: [0x0B, 0x01, 0x00, 0x00, 0x00, 0x00]
        - delay: 500ms
        # Set units to Celsius on FFF5 (SettingsData)
        - ble_client.ble_write:
            id: ibbq_client
            service_uuid: "FFF0"
            characteristic_uuid: "FFF5"
            value: [0x02, 0x00, 0x00, 0x00, 0x00, 0x00]
```

### Handling disconnects

When the iBBQ disconnects (e.g. powered off, out of range), all probes are set to NAN so Home Assistant shows them as unavailable. ESPHome's `ble_client` will automatically attempt to reconnect:

```yaml
    on_disconnect:
      then:
        - lambda: |-
            ESP_LOGD("ibbq", "Disconnected!");
            id(probe_1).publish_state(NAN);
            id(probe_2).publish_state(NAN);
            id(probe_3).publish_state(NAN);
            id(probe_4).publish_state(NAN);
            id(probe_5).publish_state(NAN);
            id(probe_6).publish_state(NAN);
```

### Receiving temperature data

A `ble_client` sensor subscribes to notifications on `FFF4`. `update_interval: never` is important -- the iBBQ only supports notify, not read, so polling would cause errors.

The lambda parses the notification payload: each probe is 2 bytes little-endian in 0.1°C units. Values >= 60000 (typically `0xFFF6`) mean no probe is connected and are published as NAN. The number of probes is determined dynamically from the payload size, so this works with 4-probe and 6-probe models alike:

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

### Probe template sensors

One template sensor per probe exposes the temperature to Home Assistant. Adjust the count to match your device:

```yaml
  - platform: template
    name: "iBBQ Probe 1"
    id: probe_1
    unit_of_measurement: "°C"
    device_class: temperature
    state_class: measurement
    accuracy_decimals: 1
  # ... repeat for probe_2 through probe_6
```

## Notes

- Requires `esp-idf` framework (not Arduino) for reliable BLE client support
- The 2s delay after connect avoids a race condition where the write fires before the GATT client is fully ready
- The iBBQ only advertises when not connected to another device -- close the Inkbird phone app before testing

## Acknowledgements

Protocol details from [runningtoy/InkBird_BLE2MQTT](https://github.com/runningtoy/InkBird_BLE2MQTT) -- thank you for working out the iBBQ GATT protocol.
