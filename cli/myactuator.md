# MyActuator CLI Sub-Commands

Practical reference for `motor_cli` MyActuator control in `motorbridge`.

> Chinese version: [myactuator.zh-CN.md](myactuator.zh-CN.md)

**Related:** [CLI Reference](reference.md) | [snippets/channel-compat.md](../snippets/channel-compat.md)

## Channel Compatibility

> **Channel compatibility:** See [snippets/channel-compat.md](../snippets/channel-compat.md).

## 1) Basics

- Vendor name: `myactuator`
- CAN IDs:
  - Tx command: `0x140 + motor_id`
  - Rx feedback: `0x240 + motor_id`
- Typical ID range: `motor_id in [1, 32]`
- Common default for ID=1:
  - `--motor-id 1`
  - `--feedback-id 0x241`

## 2) Supported `motor_cli` Modes

| Mode | Protocol Command | Description |
|---|---|---|
| `scan` | - | Probe IDs in 1..32 |
| `enable` | `0x77` | Release brake |
| `disable` | `0x80` | Shutdown motor |
| `stop` | `0x81` | Stop closed loop |
| `set-zero` | `0x64` | Set current position as encoder zero (persistent after power-cycle) |
| `status` | `0x9C` | Request status-2 |
| `current` | `0xA1` | Current closed loop |
| `vel` | `0xA2` | Speed closed loop |
| `pos` | `0xA4` | Absolute position closed loop |
| `version` | `0xB2` | Query version date |
| `mode-query` | `0x70` | Query system operating mode |

## 3) Scan

Scan for MyActuator motors in the ID range 1..32.

```bash
motor_cli --vendor myactuator --channel can0 --mode scan --start-id 1 --end-id 32
```

## 4) Enable / Disable

```bash
# Enable (release brake)
motor_cli --vendor myactuator --channel can0 --model X8 --motor-id 1 --feedback-id 0x241 \
  --mode enable --loop 1

# Disable (shutdown motor)
motor_cli --vendor myactuator --channel can0 --model X8 --motor-id 1 --feedback-id 0x241 \
  --mode disable --loop 1

# Stop closed loop
motor_cli --vendor myactuator --channel can0 --model X8 --motor-id 1 --feedback-id 0x241 \
  --mode stop --loop 1
```

## 5) Status Query

Request status-2 (`0x9C`) continuously.

```bash
motor_cli --vendor myactuator --channel can0 --model X8 --motor-id 1 --feedback-id 0x241 \
  --mode status --loop 40 --dt-ms 50
```

Status output now prints both:

- `angle=...` from status-2 (`0x9C`) -- near-turn angle
- `mt_angle=...` from multi-turn angle (`0x92`) -- for absolute-position judgement

## 6) Velocity Mode (`vel`)

Speed closed loop (`0xA2`).

```bash
# +0.5236 rad/s (~+30 deg/s)
motor_cli --vendor myactuator --channel can0 --model X8 --motor-id 1 --feedback-id 0x241 \
  --mode vel --vel 0.5236 --loop 100 --dt-ms 20
```

## 7) Position Mode (`pos`)

Absolute position closed loop (`0xA4`).

```bash
# pi rad = 180 deg, max 5.236 rad/s ~ 300 deg/s
motor_cli --vendor myactuator --channel can0 --model X8 --motor-id 1 --feedback-id 0x241 \
  --mode pos --pos 3.1416 --max-speed 5.236 --loop 1
```

## 8) Current Mode (`current`)

Current closed loop (`0xA1`).

```bash
motor_cli --vendor myactuator --channel can0 --model X8 --motor-id 1 --feedback-id 0x241 \
  --mode current --current 1.0 --loop 100 --dt-ms 20
```

## 9) Set Zero

Set current position as encoder zero. Requires power-cycle to apply persistently.

```bash
motor_cli --vendor myactuator --channel can0 --model X8 --motor-id 1 --feedback-id 0x241 \
  --mode set-zero --loop 1
```

## 10) Version and Mode Query

```bash
# Query firmware version date
motor_cli --vendor myactuator --channel can0 --model X8 --motor-id 1 --feedback-id 0x241 \
  --mode version --loop 1

# Query system operating mode
motor_cli --vendor myactuator --channel can0 --model X8 --motor-id 1 --feedback-id 0x241 \
  --mode mode-query --loop 1
```

## 11) Arguments and Units

| Argument | Type | Default | Used in | Unit |
|---|---|---|---|---|
| `--start-id` | u16 | `1` | `scan` | id |
| `--end-id` | u16 | `32` | `scan` | id |
| `--current` | f32 | `0.0` | `current` | A |
| `--vel` | f32 | `0.0` | `vel` | rad/s |
| `--pos` | f32 | `0.0` | `pos` | rad |
| `--max-speed` | f32 | `8.726646` | `pos` | rad/s |
| `--loop` | u64 | `1` | all | cycles |
| `--dt-ms` | u64 | `20` | all | ms |

## 12) Encoding Details (Current CLI)

CLI input is unified in radians/rad-s, then converted internally to MyActuator protocol units (deg/deg-s).

- `A1` current mode:
  - payload `[4..5] = int16(current / 0.01)`
- `A2` velocity mode:
  - payload `[4..7] = int32((vel_rad_s.to_degrees()) * 100)`
- `A4` absolute position mode:
  - payload `[2..3] = uint16(max_speed_rad_s.to_degrees())`
  - payload `[4..7] = int32(pos_rad.to_degrees() * 100)`

## 13) Feedback Decoding (status/command reply)

For response commands `0x9C`, `0xA1`, `0xA2`, `0xA4`:

- `data[1]`: temperature (int8, deg C)
- `data[2..3]`: current (int16 * 0.01 A)
- `data[4..5]`: speed (int16, deg/s)
- `data[6..7]`: shaft angle (int16 * 0.01 deg, near-turn angle)

For `0x92` multi-turn angle response:

- `data[4..7]`: multi-turn angle (int32 * 0.01 deg)

CLI status output now prints both:

- `angle=...` from status-2 (`0x9C`)
- `mt_angle=...` from multi-turn angle (`0x92`) for absolute-position judgement

## 14) Troubleshooting

If motor replies but does not move:

1. Query status-1 (`0x9A`) externally and check error code.
2. If error code is `0x0004`, this means **low voltage** protection.
3. Recover supply voltage, then reset (`0x76`) and release brake (`0x77`).

Typical status-1 decode:

- `data[3]`: brake released flag
- `data[4..5]`: bus voltage (`uint16 * 0.1 V`)
- `data[6..7]`: error code
