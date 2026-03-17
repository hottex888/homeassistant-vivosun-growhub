# Vivosun Integration Protocol Notes

## Scope

This document summarizes the Vivosun cloud and MQTT behavior currently implemented by this integration.

It is not intended to be a vendor-complete protocol reference. It focuses on the endpoints, topics, payload fragments, and value mappings needed by the current Home Assistant integration for:

- GrowHub controller devices
- AeroStream humidifier devices
- AeroFlux heater devices

## Architecture overview

The integration uses two communication layers:

1. REST API for authentication, device discovery, AWS identity exchange, and point-log telemetry
2. AWS IoT MQTT for device control, reported shadow state, connectivity, and live sensor updates

## REST API

### Base URL

```text
https://api-prod.next.vivosun.com
```

### Headers

Required for unauthenticated requests:

```text
Content-Type: application/json
```

Required for authenticated requests:

```text
login-token: <loginToken>
access-token: <accessToken>
```

### Standard response envelope

```json
{
  "code": 0,
  "success": true,
  "message": "success",
  "data": {}
}
```

## Authentication

### POST /user/login

Request:

```json
{
  "email": "<email>",
  "password": "<password>",
  "spAppId": "com.vivosun.android",
  "spClientId": "<uuid-v4>",
  "spSessionId": "<uuid-v4>"
}
```

Response data fields:

```json
{
  "accessToken": "<jwt>",
  "loginToken": "<jwt>",
  "refreshToken": "<jwt>",
  "userId": "153685567990966911"
}
```

### Token lifecycle

| Token | Approximate lifetime |
| --- | --- |
| `loginToken` | ~10 months |
| `accessToken` | ~3 months |
| `refreshToken` | ~10 months |
| AWS STS credentials | ~1 hour |

The integration currently performs a fresh login when needed instead of trying to use a separate refresh endpoint.

## Device discovery

### GET /iot/device/getTotalList

Representative device entry:

```json
{
  "clientId": "vivosun-VSCTLE42A-153685488534068007-153685488534068013",
  "deviceId": "153685488534068013",
  "name": "GrowHub E42A",
  "topicPrefix": "vivosun/VS_COMMON/VSCTLE42A/153685488534068007/153685488534068013",
  "onlineStatus": 1,
  "scene": {
    "sceneId": 66078
  }
}
```

Field meanings:

- `clientId`: AWS IoT thing name used for shadow topics
- `deviceId`: integration-local device key
- `topicPrefix`: MQTT prefix used for `channel/app`
- `onlineStatus`: `1` for online, `0` for offline
- `scene.sceneId`: required for point-log telemetry requests

Current integration behavior:

- devices without `clientId` are ignored
- camera-like devices are ignored
- remaining devices are classified heuristically by name and `clientId`

## AWS identity

### POST /iot/user/awsIdentity

Request:

```json
{
  "awsIdentityId": "",
  "attachPolicy": true
}
```

Relevant response fields:

```json
{
  "awsHost": "<aws-iot-endpoint>",
  "awsRegion": "us-east-2",
  "awsIdentityId": "<identity-id>",
  "awsOpenIdToken": "<token>",
  "awsPort": 443
}
```

The integration exchanges this identity data for temporary AWS credentials and uses SigV4 to sign the MQTT websocket URL.

## Telemetry

### POST /iot/data/getPointLog

Representative request:

```json
{
  "sceneId": 66078,
  "timeLevel": "ONE_MINUTE",
  "reportType": 0,
  "orderBy": "asc",
  "startTime": 1772781060,
  "endTime": 1772781733,
  "deviceId": "153685488534068013"
}
```

Parameters:

| Field | Type | Description |
| --- | --- | --- |
| `deviceId` | string | Device ID |
| `sceneId` | int | Scene ID from device discovery |
| `startTime` | int | Unix timestamp, seconds |
| `endTime` | int | Unix timestamp, seconds |
| `reportType` | int | `0` for sensor data |
| `orderBy` | string | `asc` or `dsc` |
| `timeLevel` | string | `ONE_MINUTE` |

Response data contains `iotDataLogList`, with each row carrying point-in-time telemetry.

Representative row:

```json
{
  "inTemp": 2004,
  "inHumi": 5508,
  "inVpd": 105,
  "outTemp": 1985,
  "outHumi": 5449,
  "outVpd": 105,
  "pTemp": 2150,
  "pHumi": 4875,
  "pVpd": 134,
  "waterLv": 20000,
  "coreTemp": 3839,
  "rssi": -35,
  "time": 1772781720
}
```

### Value scaling

| Key | Scaling | Example | Displayed value |
| --- | --- | --- | --- |
| `inTemp`, `outTemp`, `pTemp` | divide by 100 | `2004` | `20.04 C` |
| `inHumi`, `outHumi`, `pHumi` | divide by 100 | `5508` | `55.08 %` |
| `inVpd`, `outVpd`, `pVpd` | divide by 100 | `105` | `1.05 kPa` |
| `waterLv` | divide by 1000 | `20000` | `20.0 %` |
| `coreTemp` | divide by 100 | `3839` | `38.39 C` |
| `rssi` | no scaling | `-35` | `-35 dBm` |

Sentinel value:

- `-6666` means the sensor is unavailable or not connected

## MQTT topics

### Shadow topics

The integration uses the classic unnamed shadow:

```text
$aws/things/{thing}/shadow/get
$aws/things/{thing}/shadow/get/accepted
$aws/things/{thing}/shadow/update
$aws/things/{thing}/shadow/update/accepted
$aws/things/{thing}/shadow/update/documents
$aws/things/{thing}/shadow/update/delta
```

### Channel topic

```text
{topicPrefix}/channel/app
```

The integration routes shadow topics by `clientId` and `channel/app` topics by `topicPrefix`.

## Shadow schema

### Controller fragments

```json
{
  "light": {
    "mode": 0,
    "manu": {
      "lv": 25,
      "spec": 20
    }
  },
  "cFan": {
    "mode": 0,
    "manu": {
      "lv": 70
    },
    "osc": 0,
    "nw": 0
  },
  "dFan": {
    "mode": 0,
    "manu": {
      "lv": 60
    },
    "auto": {
      "tMin": -6666,
      "tMax": 2800,
      "hMin": -6666,
      "hMax": 7000,
      "vpdMin": -6666,
      "vpdMax": 180,
      "tStep": 100,
      "hStep": 100,
      "vpdStep": 10
    }
  },
  "connection": {
    "connected": true
  }
}
```

### Humidifier fragment

```json
{
  "hmdf": {
    "on": 1,
    "mode": 1,
    "manu": {
      "lv": 3
    },
    "targetHumi": 5500,
    "waterWarn": 0
  }
}
```

### Heater fragment

```json
{
  "heat": {
    "on": 1,
    "mode": 1,
    "state": 1,
    "manu": {
      "lv": 7
    },
    "targetTemp": 2350
  }
}
```

### Channel/app sensor keys

Currently parsed keys are:

- `inTemp`, `inHumi`, `inVpd`
- `outTemp`, `outHumi`, `outVpd`
- `pTemp`, `pHumi`, `pVpd`
- `waterLv`, `coreTemp`, `rssi`

## Control payloads

All write-path control updates use `state.desired` payloads.

Representative forms:

```json
{
  "state": {
    "desired": {
      "light": {
        "mode": 0,
        "manu": {
          "lv": 25
        }
      }
    }
  }
}
```

```json
{
  "state": {
    "desired": {
      "hmdf": {
        "targetHumi": 5500
      }
    }
  }
}
```

```json
{
  "state": {
    "desired": {
      "heat": {
        "targetTemp": 2350
      }
    }
  }
}
```

## Device-specific control mappings

### Light

- `0` means off
- `1..24` are clamped to `25`
- `25..100` are passed through

### Duct fan mapping

| App level | Shadow `manu.lv` |
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

### Circulation fan mapping

| App level | Shadow `manu.lv` |
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
| Natural Wind | 200 |

Additional flags:

- circulation fan night mode uses `nw`
- circulation fan oscillation uses `osc`

### Humidifier and heater targets

- `targetHumi` is stored as an integer scaled by `100`
- `targetTemp` is stored as an integer scaled by `100`

## Notes

- `update/delta` traffic is intentionally not treated as live entity state
- point-log polling still matters even when `channel/app` traffic is present
- device classification is heuristic and based on currently known naming patterns
