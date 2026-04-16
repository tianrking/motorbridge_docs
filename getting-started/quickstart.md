# Python Binding Quickstart (pip-first)

This is a beginner-first quickstart for users who install from pip.

Goal: install package → run scan → run a motor example in minutes.

## 1) Install

```bash
python3 -m pip install motorbridge
```

If you want to test pre-release package from TestPyPI:

```bash
python3 -m pip install -i https://test.pypi.org/simple/ motorbridge==<version>
```

## 2) Hardware and Channel

- Linux SocketCAN channel examples: `can0`, `can1`, `slcan0`
- Windows PCAN channel examples: `can0@1000000`, `can1@1000000`
- Ensure only one sender is writing to the bus while testing.

> **Channel compatibility details:** See [snippets/channel-compat.md](../snippets/channel-compat.md)

### Which transport should I use?

- `TRANSPORT = "auto"` or `"socketcan"`:
  use normal CAN path (`CHANNEL` is used).
- `TRANSPORT = "dm-serial"`:
  use Damiao serial bridge (`SERIAL_PORT` / `SERIAL_BAUD` are used), Damiao only.

## 3) Quick Commands (no source checkout required)

```bash
# Unified scan from installed CLI
motorbridge-cli scan --vendor all --channel can0 --start-id 1 --end-id 255

# Single Damiao control (example IDs)
motorbridge-cli run --vendor damiao --channel can0 --model 4340P --motor-id 0x01 --feedback-id 0x11 \
  --mode pos-vel --pos 1.0 --vlim 1.0 --loop 60 --dt-ms 20
```

## 4) Config constants (simple meaning)

- `TRANSPORT`: `auto/socketcan/dm-serial`
- `CHANNEL`: CAN interface name (`can0`, `slcan0`, `can0@1000000`, ...)
- `VENDOR`: scan target vendor (`all` is most common)
- `MOTOR_ID` / `FEEDBACK_ID`: motor command/feedback IDs
- `MODEL`: motor model string
- `TARGET_POS` / `POS`: target position in radians
- `V_LIMIT`: velocity limit for position mode
- `LOOP`: send loop count
- `DT_MS`: control period in ms
- `SERIAL_PORT` / `SERIAL_BAUD`: used only for `dm-serial`

## 5) Course Series (workflow-first)

If you want a strict real-world workflow course, use `courses/`:

| Course | Topic |
|--------|-------|
| [00-enable-and-status](courses/00-enable-and-status.md) | Enable + status query |
| [01-scan](courses/01-scan.md) | Scan devices |
| [02-register-rw](courses/02-register-rw.md) | Register read/write |
| [03-mode-switch-method](courses/03-mode-switch-method.md) | Mode switching |
| [04-mode-mit](courses/04-mode-mit.md) | MIT mode |
| [05-mode-pos-vel](courses/05-mode-pos-vel.md) | POS_VEL mode |
| [06-mode-vel](courses/06-mode-vel.md) | VEL mode |
| [07-mode-force-pos](courses/07-mode-force-pos.md) | FORCE_POS mode |
| [08-mode-mixed-switch](courses/08-mode-mixed-switch.md) | Mixed mode switching |
| [09-multi-motor](courses/09-multi-motor.md) | Multi-motor control |

Example:

```bash
python3 courses/00-enable-and-status.py
python3 courses/01-scan.py
python3 courses/03-mode-switch-method.py
```

## 6) Common Issues

- `os error 105`: bus TX is too fast or another process is sending; increase `--dt-ms` to 30/50.
- no motor response: verify CAN wiring, bitrate, motor/feedback IDs.
- `slcan` users: bring up `slcan0` before running examples.

## 7) `poll_feedback_once()` version note

- `<= v0.1.6`: keep manual `poll_feedback_once()` in status-query scripts.
- `v0.1.7+`: background polling is enabled by default, so manual poll is usually optional.
- Course scripts keep `poll_feedback_once()` for backward-compatible teaching style.

## 8) What to read next

- Python API overview: [sdk/python/reference.md](../sdk/python/reference.md)
- CLI full docs: [cli/reference.md](../cli/reference.md)
- Architecture: [architecture/overview.md](../architecture/overview.md)
