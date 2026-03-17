# Architecture

## Summary

The integration uses a hybrid runtime model:

- REST bootstrap for login, device discovery, and AWS identity exchange
- MQTT over AWS IoT websocket for control, reported shadow state, connectivity, and live `channel/app` sensor traffic
- REST point-log polling for telemetry backfill and periodic refresh

This split exists because control and reported state are available through AWS IoT shadow topics, while point-log history is still needed to populate and refresh the current sensor snapshot for each device.

## Bootstrap sequence

1. `POST /user/login`
2. `GET /iot/device/getTotalList`
3. classify all discovered devices
4. filter to MQTT-capable non-camera devices
5. `POST /iot/user/awsIdentity`
6. Cognito credential exchange
7. MQTT websocket connect
8. initial shadow `get` for each device
9. initial point-log refresh for each device

The coordinator owns this sequence in [coordinator.py](../custom_components/vivosun_growhub/coordinator.py).

## Device selection

The integration now keeps all supported MQTT-capable devices discovered for the account.

Selection rules:

- devices without `client_id` are filtered out
- camera devices are filtered out
- remaining devices are sorted by `(device_id, client_id, topic_prefix)`
- the first controller in that sorted set is treated as the primary controller for controller-specific entities such as `light` and `fan`

Device typing is currently inferred heuristically from device name and `client_id` patterns:

- `controller` for GrowHub-like devices
- `humidifier` for AeroStream-like devices
- `heater` for AeroFlux-like devices
- `camera` for GrowCam-like devices

## Coordinator snapshot

Entities consume a per-device snapshot shaped like:

- `devices`
- `shadows`
- `sensors`
- `mqtt_connected`

`devices`, `shadows`, and `sensors` are keyed by `device_id`.

## MQTT path

### Used for

- light state and control
- circulation fan state and control
- duct fan state and control
- humidifier state and control
- heater state and control
- connection state
- live sensor updates from `channel/app`

### Topics

The integration uses the classic unnamed shadow:

- `$aws/things/{thing}/shadow/get`
- `$aws/things/{thing}/shadow/get/accepted`
- `$aws/things/{thing}/shadow/update`
- `$aws/things/{thing}/shadow/update/accepted`
- `$aws/things/{thing}/shadow/update/documents`
- `$aws/things/{thing}/shadow/update/delta`

It also subscribes to:

- `{topicPrefix}/channel/app`

### Important behavior

`update/delta` is not surfaced as live entity state.

Reason:

- delta can temporarily reflect desired-versus-reported drift
- treating delta as live state caused false UI snaps, including light values jumping back during state convergence

Only reported, accepted, and documents payloads are merged into the visible shadow snapshot. `channel/app` payloads update the sensor snapshot directly.

## Sensor telemetry path

### Used for

- inside temperature, humidity, and VPD
- outside temperature, humidity, and VPD
- probe temperature, humidity, and VPD
- water level
- optional core temperature and RSSI

### REST endpoint

- `POST /iot/data/getPointLog`

### Polling model

- a recent window is requested per discovered device
- the newest row from `iotDataLogList` becomes that device's current sensor snapshot
- the coordinator refresh interval is 90 seconds

### MQTT live updates

When available, `channel/app` payloads update the in-memory sensor snapshot between poll cycles.

## Platform model

- `light.py` and `fan.py` target the primary controller device
- `sensor.py` creates entities based on `device_type`
- `binary_sensor.py` creates one connectivity entity per discovered device
- `humidifier.py` binds humidifier shadow state to probe humidity telemetry
- `climate.py` binds heater shadow state to probe temperature telemetry

## Device-specific mappings

### Light

- `0` means off
- `1..24` are clamped to `25`
- `25..100` pass through unchanged

### Duct fan

Home Assistant exposes a 10-step speed model. The device shadow uses non-linear `manu.lv` values.

| App level | Shadow value |
| --- | --- |
| 0 | 0 |
| 1 | 30 |
| 2 | 35 |
| 3 | 40 |
| 4 | 50 |
| 5 | 60 |
| 6 | 70 |
| 7 | 80 |
| 8 | 85 |
| 9 | 90 |
| 10 | 100 |

### Circulation fan

| App level | Shadow value |
| --- | --- |
| 0 | 0 |
| 1 | 44 |
| 2 | 51 |
| 3 | 60 |
| 4 | 64 |
| 5 | 70 |
| 6 | 75 |
| 7 | 80 |
| 8 | 85 |
| 9 | 90 |
| 10 | 100 |

Special modes:

- `natural_wind` is represented by `lv = 200`
- `night` is represented by `nw = 1`

### Humidifier and heater

- manual/auto mode is controlled through the shadow `mode` field
- humidifier auto target humidity is stored as `targetHumi`, scaled by `100`
- heater auto target temperature is stored as `targetTemp`, scaled by `100`

## Failure handling

### MQTT reconnect

The coordinator supervises the websocket session and reconnects when:

- the session drops
- AWS credentials approach expiry
- a full reauthentication is needed

### Credential refresh

AWS credentials are refreshed before expiry using the configured skew window. If refresh fails due to auth expiry, the coordinator performs a full login, identity exchange, and reconnect.
