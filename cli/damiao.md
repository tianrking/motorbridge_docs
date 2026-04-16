# Damiao CLI Sub-Commands

This page is a practical reference for all commonly adjustable control/configuration parameters currently available in `motorbridge` for Damiao motors via the `motor_cli` binary.

> Chinese version: [damiao.zh-CN.md](damiao.zh-CN.md)

**Related:** [CLI Reference](reference.md) | [snippets/channel-compat.md](../snippets/channel-compat.md)

## Channel Compatibility

> **Channel compatibility:** See [snippets/channel-compat.md](../snippets/channel-compat.md).

## Supported Modes

| Mode | Description |
|---|---|
| `scan` | Probe motors in an ID range |
| `enable` | Enable motor (exit damping) |
| `disable` | Disable motor (enter damping) |
| `mit` | MIT impedance control (`pos/vel/kp/kd/tau`) |
| `pos-vel` | Position-velocity control (`pos/vlim`) |
| `vel` | Velocity control (`vel`) |
| `force-pos` | Force-position control (`pos/vlim/ratio`) |

## 1) Common Device Parameters

| Parameter | Meaning | Typical value |
|---|---|---|
| `channel` | CAN interface name | `can0` |
| `model` | Motor model string (`4310`, `4340P`, etc.) | matches real device |
| `motor-id` | Command ID (`ESC_ID`) | e.g. `0x01` |
| `feedback-id` | Feedback ID (`MST_ID`) | e.g. `0x11` |
| `loop` | Number of send cycles | `100` |
| `dt-ms` | Send interval per cycle (ms) | `20` |

## 2) Scan

Scan a range of CAN IDs to discover Damiao motors.

### Arguments

| Argument | Type | Default | Notes |
|---|---|---|---|
| `--start-id` | u16 | `1` | Scan start, must be 1..255 |
| `--end-id` | u16 | `255` | Scan end, must be 1..255 |

### Scan Behavior Details

- The scanner is model-agnostic in practice: it internally tries a built-in model-hint list.
- For each candidate ID, it also tries multiple feedback-ID hints: inferred (`id+0x10`), user `--feedback-id`, `0x11`, `0x17`.
- Detection first attempts register reads (RID 21/22/23), then feedback fallback.

### Example

```bash
motor_cli \
  --vendor damiao --channel can0 --mode scan --start-id 1 --end-id 16
# [STD-CAN]
```

## 3) Enable / Disable

Enable or disable (damping) a motor.

```bash
# Enable
motor_cli \
  --vendor damiao --channel can0 --model 4310 --motor-id 0x04 --feedback-id 0x14 \
  --mode enable --loop 1

# Disable
motor_cli \
  --vendor damiao --channel can0 --model 4310 --motor-id 0x04 --feedback-id 0x14 \
  --mode disable --loop 1
```

No extra control arguments are required for `enable`/`disable`.

## 4) MIT Mode

Command fields:

- `pos`: target position
- `vel`: target velocity
- `kp`: position stiffness gain
- `kd`: velocity damping gain
- `tau`: feedforward torque

### Arguments

| Argument | Type | Default |
|---|---|---|
| `--pos` | f32 | `0` |
| `--vel` | f32 | `0` |
| `--kp` | f32 | `2` |
| `--kd` | f32 | `1` |
| `--tau` | f32 | `0` |

### Examples

```bash
# MIT control over standard CAN
motor_cli \
  --vendor damiao --channel can0 --model 4310 --motor-id 0x04 --feedback-id 0x14 \
  --mode mit --pos 1.57 --vel 2.0 --kp 35 --kd 1.2 --tau 0.3 --loop 120 --dt-ms 20
# [STD-CAN]

# MIT control via Damiao serial bridge
motor_cli \
  --vendor damiao --transport dm-serial --serial-port /dev/ttyACM1 --serial-baud 921600 \
  --model 4310 --motor-id 0x04 --feedback-id 0x14 \
  --mode mit --verify-model 0 --ensure-mode 0 \
  --pos 1.0 --vel 0 --kp 2 --kd 1 --tau 0 --loop 80 --dt-ms 20
# [DM-SERIAL]

# MIT control via dedicated CAN-FD transport
motor_cli \
  --vendor damiao --transport socketcanfd --channel can0 \
  --model 4310 --motor-id 0x04 --feedback-id 0x14 \
  --mode mit --verify-model 0 --ensure-mode 0 \
  --pos 0.5 --vel 0 --kp 20 --kd 1 --tau 0 --loop 80 --dt-ms 20
# [CAN-FD]
```

## 5) POS_VEL Mode

Command fields:

- `pos`: target position
- `vlim`: velocity limit

### Arguments

| Argument | Type | Default |
|---|---|---|
| `--pos` | f32 | `0` |
| `--vlim` | f32 | `1.0` |

### Example

```bash
motor_cli \
  --vendor damiao --channel can0 --model 4340P --motor-id 0x01 --feedback-id 0x11 \
  --mode pos-vel --pos 3.10 --vlim 1.50 --loop 300 --dt-ms 20
# [STD-CAN]
```

## 6) VEL Mode

Command field:

- `vel`: target velocity

### Arguments

| Argument | Type | Default |
|---|---|---|
| `--vel` | f32 | `0` |

### Example

```bash
motor_cli \
  --vendor damiao --channel can0 --model 4340P --motor-id 0x01 --feedback-id 0x11 \
  --mode vel --vel 0.5 --loop 100 --dt-ms 20
# [STD-CAN]
```

## 7) FORCE_POS Mode

Command fields:

- `pos`: target position
- `vlim`: velocity limit
- `ratio`: torque limit ratio

### Arguments

| Argument | Type | Default |
|---|---|---|
| `--pos` | f32 | `0` |
| `--vlim` | f32 | `1.0` |
| `--ratio` | f32 | `0.1` |

### Example

```bash
motor_cli \
  --vendor damiao --channel can0 --model 4340P --motor-id 0x01 --feedback-id 0x11 \
  --mode force-pos --pos 0.8 --vlim 2.0 --ratio 0.3 --loop 100 --dt-ms 20
# [STD-CAN]
```

## 8) Mode Selection Register (`CTRL_MODE`)

Register:

- `rid=10` (`CTRL_MODE`), values:
  - `1=MIT`
  - `2=POS_VEL`
  - `3=VEL`
  - `4=FORCE_POS`

## 9) Damiao Extra Arguments

| Argument | Type | Default | Used In | Notes |
|---|---|---|---|---|
| `--verify-model` | `0/1` | `1` | non-scan | Verify PMAX/VMAX/TMAX matches `--model` |
| `--verify-timeout-ms` | u64 | `500` | non-scan | Register read timeout for model handshake |
| `--verify-tol` | f32 | `0.2` | non-scan | Model limit tolerance |
| `--set-motor-id` | u16 opt | none | id-set flow | Write ESC_ID (RID 8) |
| `--set-feedback-id` | u16 opt | none | id-set flow | Write MST_ID (RID 7) |
| `--store` | `0/1` | `1` | id-set flow | Persist parameters |
| `--verify-id` | `0/1` | `1` | id-set flow | Re-read RID7/RID8 and verify |

## 10) High-impact Registers (Priority)

These have major control impact and should be tuned carefully.

| RID | Name | Type | Meaning |
|---|---|---|---|
| `21` | `PMAX` | `f32` | Position mapping range |
| `22` | `VMAX` | `f32` | Velocity mapping range |
| `23` | `TMAX` | `f32` | Torque mapping range |
| `25` | `KP_ASR` | `f32` | Speed loop Kp |
| `26` | `KI_ASR` | `f32` | Speed loop Ki |
| `27` | `KP_APR` | `f32` | Position loop Kp |
| `28` | `KI_APR` | `f32` | Position loop Ki |
| `4` | `ACC` | `f32` | Acceleration |
| `5` | `DEC` | `f32` | Deceleration |
| `6` | `MAX_SPD` | `f32` | Maximum speed |
| `9` | `TIMEOUT` | `u32` | Communication timeout register |

## 11) Protection Registers

| RID | Name | Type | Meaning |
|---|---|---|---|
| `0` | `UV_Value` | `f32` | Under-voltage threshold |
| `2` | `OT_Value` | `f32` | Over-temperature threshold |
| `3` | `OC_Value` | `f32` | Over-current threshold |
| `29` | `OV_Value` | `f32` | Over-voltage threshold |

## 12) How to Read/Write Parameters

Recommended write workflow:

1. `get_register` read old value
2. `write_register` write new value
3. `get_register` read back
4. `store_parameters` persist

### Python SDK API Example

```python
from motorbridge import Controller

with Controller("can0") as ctrl:
    m = ctrl.add_damiao_motor(0x01, 0x11, "4340P")
    old = m.get_register_f32(22, 1000)
    print("old VMAX", old)
    m.write_register_f32(22, old * 0.9)
    new = m.get_register_f32(22, 1000)
    print("new VMAX", new)
    m.store_parameters()
    m.close()
```

### WS Gateway Command Examples

```json
{"op":"get_register_f32","rid":22,"timeout_ms":1000}
{"op":"write_register_f32","rid":22,"value":25.0}
{"op":"store_parameters"}
```

## 13) ID and Calibration Parameters

- `rid=8` (`ESC_ID`): command ID
- `rid=7` (`MST_ID`): feedback ID

### Update ID Example

```bash
# Update ID and persist
motor_cli \
  --vendor damiao --channel can0 --model 4310 --motor-id 0x01 --feedback-id 0x11 \
  --set-motor-id 0x04 --set-feedback-id 0x14 --store 1 --verify-id 1
```

### Tools

- Rust CLI: `motor_cli --vendor damiao --mode scan ...` and `--set-motor-id/--set-feedback-id --store --verify-id`
- Python: `motorbridge-cli scan/id-set/id-dump`
- WS: `scan`, `set_id`, `verify`

## 14) Safety Notes

- Tune one parameter group at a time.
- Use small steps, verify each change.
- Keep safe mechanical load and emergency stop ready.
- For protection thresholds (`0/2/3/29`), avoid aggressive changes without hardware margin analysis.

## 15) Gripper Motor Calibration

Damiao motors use a single-turn encoder (position range approx. +/-PMAX rad). **Zero position is NOT retained across power cycles.** When a motor is used as a gripper, calibrate the zero position after each power-on:

1. Use MIT mode with low `kp` and `tau=0` to move toward the mechanical limit -- the motor stops softly on contact.
2. Wait until feedback shows `vel ~ 0` and `pos` is stable, then run `--mode set-zero`.
3. After calibration, use MIT mode for closing (force-limited, safe) and pos-vel mode for opening (fast, precise).

For full command examples, see the Operation Manual (Section 9).
