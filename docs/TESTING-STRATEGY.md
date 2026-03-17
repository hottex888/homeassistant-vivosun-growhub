# Testing Strategy

## Scope

This integration is tested as a hybrid cloud integration with multi-device behavior:

- REST bootstrap and point-log telemetry polling
- MQTT websocket transport, shadow updates, and `channel/app` sensor updates
- Home Assistant entities, config flow, options flow, and diagnostics

## Test layers

### Unit tests

Focused tests cover:

- REST API parsing and error handling
- Cognito and SigV4 credential flow
- MQTT packet encoding and decoding
- shadow parsing and desired payload construction
- light and fan mapping helpers
- heater and humidifier payload builders

### Integration tests

Coordinator and entity tests cover:

- bootstrap sequence
- device discovery and per-device selection
- reconnect behavior
- shadow state merging
- `channel/app` sensor updates
- point-log sensor refresh
- config flow and options flow
- diagnostics redaction

### Platform tests

Platform-specific coverage verifies:

- controller light and fan behavior
- humidifier state mapping and control payloads
- climate state mapping, Fahrenheit conversion, and target-setting behavior
- sensor creation by device type
- connectivity entity behavior

### Smoke tests

Smoke coverage verifies:

- full setup path
- control roundtrips for light and fans
- sensor population from the hybrid runtime model
- unload and reload behavior

## Current commands

### Full test suite

```bash
.venv/bin/pytest -q
```

### Lint

```bash
.venv/bin/ruff check .
```

### Type checking

```bash
.venv/bin/python -m mypy --explicit-package-bases custom_components/vivosun_growhub
```

## CI expectations

CI should fail on:

- test regressions
- lint failures
- integration package type errors
- HACS validation failures
- hassfest validation failures

Fresh CI bootstrap also needs `pycares` available because `tests/conftest.py` imports it directly.

## Manual verification before release

1. Manual install into Home Assistant
2. Config flow setup with a real account
3. Light control
4. Circulation fan control including `night` and `natural_wind`
5. Duct fan control including low-speed default behavior and auto-threshold updates
6. Humidifier control and probe humidity telemetry
7. Heater control and probe temperature telemetry
8. Climate and probe sensor population after poll refresh and MQTT updates
9. Diagnostics export
10. Cold restart and reconnect verification
