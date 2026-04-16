# Python SDK Reference

This is the comprehensive reference for the `motorbridge` Python SDK, covering installation, the `Controller` and `Motor` classes, vendor-specific APIs, state structures, and error handling.

> Chinese version: [reference.zh-CN.md](reference.zh-CN.md)

---

## Table of Contents

1. [Installation](#1-installation)
2. [Channel Format](#2-channel-format)
3. [Controller Class](#3-controller-class)
4. [Motor Class](#4-motor-class)
5. [State and Data Structures](#5-state-and-data-structures)
6. [Damiao-Specific API](#6-damiao-specific-api)
7. [RobStride-Specific API](#7-robstride-specific-api)
8. [Error Handling](#8-error-handling)
9. [Unified Mode Mapping](#9-unified-mode-mapping)
10. [Example Programs](#10-example-programs)

---

## 1. Installation

### 1.1 pip Install (Recommended)

```bash
pip install motorbridge
```

The published wheel includes the `motor_abi` shared library and the `ws_gateway` binary for your platform.

After install, the gateway binary is typically at:
`.../site-packages/motorbridge/bin/ws_gateway` (or `ws_gateway.exe` on Windows).

Gateway launch command (added to PATH by pip):

```bash
motorbridge-gateway -- --bind 127.0.0.1:9002 ...
```

Security notes for the gateway:
- Keep loopback bind (`127.0.0.1`) for local usage.
- If you bind to non-loopback addresses (`0.0.0.0` or host IP), export `MOTORBRIDGE_WS_TOKEN` before launch.
- Clients must pass the token in `x-motorbridge-token` or `Authorization: Bearer ...`.

macOS runtime note (only if you see dynamic library load errors):

```bash
GW="$(python3 -c "import motorbridge, pathlib; print(pathlib.Path(motorbridge.__file__).resolve().parent/'bin'/'ws_gateway')")"
PKG_DIR="$(python3 -c "import motorbridge, pathlib; print(pathlib.Path(motorbridge.__file__).resolve().parent)")"
DYLD_LIBRARY_PATH="$PKG_DIR/lib:${DYLD_LIBRARY_PATH:-}" "$GW" --bind 127.0.0.1:9002 --vendor damiao --channel can0 --model auto --motor-id 0x01 --feedback-id 0x11 --dt-ms 20
```

### 1.2 Local Build from Source

Build the ABI shared library first:

```bash
cd motorbridge
cargo build -p motor_abi --release
```

Then set environment variables:

```bash
export PYTHONPATH=bindings/python/src
export LD_LIBRARY_PATH=$PWD/target/release:${LD_LIBRARY_PATH}
```

### 1.3 Local Wheel Build

Linux / macOS:

```bash
pip install --user wheel
export MOTORBRIDGE_LIB=$PWD/target/release/libmotor_abi.so
pip wheel --no-build-isolation bindings/python -w bindings/python/dist
pip install bindings/python/dist/motorbridge-*.whl
```

Windows:

```cmd
python -m pip install --user wheel
set MOTORBRIDGE_LIB=%CD%\target\release\motor_abi.dll
set MOTORBRIDGE_WS_GATEWAY_BIN=%CD%\target\release\ws_gateway.exe
python -m pip wheel --no-build-isolation bindings/python -w bindings/python/dist
python -m pip install bindings/python/dist/motorbridge-*.whl
```

### 1.4 Library Search Path

The SDK searches for the `motor_abi` shared library in the following order:

1. `MOTORBRIDGE_LIB` environment variable (if set).
2. `motorbridge/lib/` inside the installed package.
3. `target/release/` relative to the repository root (4 levels up from `abi.py`).
4. `target/release/` relative to the current working directory.
5. System library search via `ctypes.util.find_library("motor_abi")`.

If none of these paths contain the library, an `AbiLoadError` is raised.

---

## 2. Channel Format

For channel compatibility details (PCAN, slcan, CAN-FD, Damiao Serial Bridge), see [channel-compat.md](../../snippets/channel-compat.md).

Summary:

- **Linux SocketCAN**: use interface names directly: `can0`, `can1`, `slcan0`. Do not append `@bitrate` (e.g., `can0@1000000` is invalid on Linux).
- **USB-serial CAN adapters**: bring up `slcan0` first:
  ```bash
  sudo slcand -o -c -s8 /dev/ttyUSB0 slcan0 && sudo ip link set slcan0 up
  ```
- **CAN-FD**: use `Controller.from_socketcanfd(channel)` in the Python SDK, or `--transport socketcanfd` in the CLI. Required for Hexfellow motors.
- **Damiao Serial Bridge**: use `Controller.from_dm_serial(serial_port, baud)`. Only available for Damiao motors.
- **Windows (PCAN)**: `can0`/`can1` map to `PCAN_USBBUS1`/`PCAN_USBBUS2`. The `@bitrate` suffix is supported (e.g., `can0@1000000`).

---

## 3. Controller Class

The `Controller` manages a CAN bus (or serial bridge) and the motors attached to it.

### 3.1 Constructors

#### `Controller(channel: str = "can0")`

Creates a controller using standard SocketCAN (or PCAN on Windows).

```python
from motorbridge import Controller

ctrl = Controller("can0")
```

Raises `CallError` if the bus cannot be opened.

#### `Controller.from_socketcanfd(channel: str = "can0")`

Creates a controller using CAN-FD transport. Required for Hexfellow motors.

```python
ctrl = Controller.from_socketcanfd("can0")
```

#### `Controller.from_dm_serial(serial_port: str = "/dev/ttyACM0", baud: int = 921600)`

Creates a controller using the Damiao serial bridge. Only works with Damiao motors.

```python
ctrl = Controller.from_dm_serial("/dev/ttyACM0", 921600)
```

### 3.2 Lifecycle Methods

| Method | Signature | Description |
|---|---|---|
| `close` | `close() -> None` | Releases the Python-side handle. Call after `shutdown`. |
| `shutdown` | `shutdown() -> None` | Shuts down the controller and performs cleanup (including device processing). Recommended for normal exit. |
| `close_bus` | `close_bus() -> None` | Closes only the bus layer. For special manual lifecycle management. Does not replace `shutdown`. |
| Context manager | `__enter__() -> Controller`, `__exit__(exc_type, exc, tb) -> None` | On exit, calls `shutdown()` then `close()`. |

Usage with context manager:

```python
with Controller("can0") as ctrl:
    # ... work with ctrl ...
    pass
# shutdown() and close() are called automatically
```

### 3.3 Broadcast Control

| Method | Signature | Description |
|---|---|---|
| `enable_all` | `enable_all() -> None` | Broadcast enable to all motors added to this controller. Recommended to wait 100--300 ms after calling. |
| `disable_all` | `disable_all() -> None` | Broadcast disable to all motors. Call before ending your program. |
| `poll_feedback_once` | `poll_feedback_once() -> None` | Manually consumes one receive queue cycle and updates the state cache. Useful with `request_feedback` for an immediate refresh. |

### 3.4 Adding Motors

| Method | Signature | Description |
|---|---|---|
| `add_damiao_motor` | `add_damiao_motor(motor_id: int, feedback_id: int, model: str) -> Motor` | Adds a Damiao motor. `model` examples: `"4310"`, `"4340P"`. |
| `add_myactuator_motor` | `add_myactuator_motor(motor_id: int, feedback_id: int, model: str) -> Motor` | Adds a MyActuator motor. `model` example: `"X8"`. |
| `add_robstride_motor` | `add_robstride_motor(motor_id: int, feedback_id: int, model: str) -> Motor` | Adds a RobStride motor. `model` example: `"rs-06"`. Default feedback ID: `0xFD`. |
| `add_hexfellow_motor` | `add_hexfellow_motor(motor_id: int, feedback_id: int, model: str) -> Motor` | Adds a Hexfellow motor. Requires CAN-FD transport. |
| `add_hightorque_motor` | `add_hightorque_motor(motor_id: int, feedback_id: int, model: str) -> Motor` | Adds a HighTorque motor. |

All `add_*_motor` methods raise `CallError` if the motor cannot be created.

### 3.5 Controller Method Reference Table

| Method | Purpose | Typical Sequence | Parameter Guidance |
|---|---|---|---|
| `Controller(channel)` | Standard CAN controller | `add_*_motor` after | `channel` e.g. `can0` |
| `Controller.from_dm_serial(port, baud)` | Damiao serial bridge | `add_damiao_motor` after | `port` e.g. `/dev/ttyACM0`; `baud` typically `921600` |
| `Controller.from_socketcanfd(channel)` | CAN-FD controller | `add_hexfellow_motor` after | `channel` e.g. `can0` |
| `add_damiao_motor(mid, fid, model)` | Bind a Damiao motor handle | `enable_all` / `ensure_mode` / `send_*` after | `motor_id` commonly `0x01--0x7F`; `feedback_id` commonly `motor_id + 0x10`; `model` e.g. `4310`/`4340P` |
| `enable_all()` | Broadcast enable | Before `ensure_mode` | Wait 100--300 ms after |
| `disable_all()` | Broadcast disable | End of program | No constraints |
| `poll_feedback_once()` | Manual feedback refresh | With `request_feedback` | Not required when background polling is active |
| `shutdown()` | Full controller shutdown | Normal exit | Also called by `__exit__` |
| `close_bus()` | Close bus layer only | Special manual lifecycle | Does not replace `shutdown` |
| `close()` | Release Python handle | After `shutdown` | Also called by `__exit__` |

---

## 4. Motor Class

The `Motor` class represents a single motor attached to a controller. All motors (regardless of vendor) share the same base API.

### 4.1 Lifecycle

| Method | Signature | Description |
|---|---|---|
| `enable` | `enable() -> None` | Enables the motor. Wait 100--300 ms after calling. |
| `disable` | `disable() -> None` | Disables the motor. Call before ending control. |
| `close` | `close() -> None` | Releases the motor handle. |

### 4.2 Mode Switching

#### `ensure_mode(mode: Mode, timeout_ms: int = 1000) -> None`

Switches the motor to the specified control mode and verifies the switch completed within `timeout_ms`.

Parameters:
- `mode`: one of `Mode.MIT`, `Mode.POS_VEL`, `Mode.VEL`, `Mode.FORCE_POS`.
- `timeout_ms`: verification timeout in milliseconds.
  - Standard CAN: 150--300 ms recommended.
  - dm-serial: 200--500 ms recommended (commonly 300).

### 4.3 Control Commands

All control commands assume the motor is enabled and the correct mode has been set via `ensure_mode`.

#### `send_mit(pos, vel, kp, kd, tau) -> None`

MIT (impedance) mode control. Five-parameter hybrid control.

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `pos` | `float` | rad | Target position |
| `vel` | `float` | rad/s | Target velocity |
| `kp` | `float` | dimensionless | Position stiffness gain (>= 0) |
| `kd` | `float` | dimensionless | Velocity damping gain (>= 0) |
| `tau` | `float` | Nm | Feedforward torque |

Typical starting values: `kp=2--10`, `kd=0.05--0.5`, `tau=0.1--1.0`.

#### `send_pos_vel(pos, vlim) -> None`

Position control with velocity limit.

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `pos` | `float` | rad | Target position |
| `vlim` | `float` | rad/s | Velocity limit (> 0, start from 0.5--2.0) |

#### `send_vel(vel) -> None`

Pure velocity control.

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `vel` | `float` | rad/s | Target velocity (start from small values) |

#### `send_force_pos(pos, vlim, ratio) -> None`

Position control with torque limit ratio.

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `pos` | `float` | rad | Target position |
| `vlim` | `float` | rad/s | Velocity limit (> 0) |
| `ratio` | `float` | 0--1 | Torque/current limit ratio (commonly 0.1--0.5). This is not precise constant-torque control. |

### 4.4 Feedback and State

#### `request_feedback() -> None`

Actively requests one frame of feedback from the motor.

#### `poll_feedback_once()` (on Controller)

Consumes one receive cycle and updates the internal state cache. Use together with `request_feedback` and `get_state` for the freshest readings.

#### `get_state() -> MotorState | None`

Returns the most recently cached motor state. Returns `None` if no feedback has been received yet. For a more reliable reading, call `request_feedback()` + `poll_feedback_once()` before `get_state()`.

### 4.5 Maintenance Operations

| Method | Signature | Description |
|---|---|---|
| `clear_error` | `clear_error() -> None` | Clears the motor fault state. Call before `enable`/`ensure_mode` during maintenance. |
| `set_zero_position` | `set_zero_position() -> None` | Sets the current position as the zero-point reference. This does **not** move the motor to position 0. The core applies a fixed 20 ms settle time internally; there is no `ms` parameter in the Python API. **Project rule: always call `disable()` before `set_zero_position()`.** |
| `set_can_timeout_ms` | `set_can_timeout_ms(timeout_ms: int) -> None` | Sets the CAN timeout parameter on the motor. `timeout_ms > 0`, commonly 100--2000. |
| `store_parameters` | `store_parameters() -> None` | Persists current register values to non-volatile storage. Call after confirming register values are correct. |

### 4.6 Register Access (Generic)

These methods work for all motor types that support register-based parameter access.

| Method | Signature | Description |
|---|---|---|
| `write_register_f32` | `write_register_f32(rid: int, value: float) -> None` | Writes a `float` value to the specified register. |
| `write_register_u32` | `write_register_u32(rid: int, value: int) -> None` | Writes an unsigned 32-bit integer value to the specified register. |
| `get_register_f32` | `get_register_f32(rid: int, timeout_ms: int = 1000) -> float` | Reads a `float` register. `timeout_ms` commonly 300--1000. |
| `get_register_u32` | `get_register_u32(rid: int, timeout_ms: int = 1000) -> int` | Reads an unsigned 32-bit integer register. `timeout_ms` commonly 300--1000. |

### 4.7 Motor Method Reference Table

| Method | Purpose | Typical Sequence | Parameter Guidance |
|---|---|---|---|
| `enable()` | Single motor enable | After `clear_error`, before control | Wait 100--300 ms |
| `disable()` | Single motor disable | End of control flow | No constraints |
| `clear_error()` | Clear fault state | Before `enable`/`ensure_mode` | Recommended in maintenance |
| `set_zero_position()` | Set current position as zero | **Must `disable` first** | No `ms` param; core uses fixed 20 ms |
| `ensure_mode(mode, timeout_ms)` | Switch + verify control mode | Before `send_*` | Standard CAN: 150--300; dm-serial: 200--500 |
| `send_mit(pos, vel, kp, kd, tau)` | MIT impedance control | After `ensure_mode(Mode.MIT)` | `pos` rad; `vel` rad/s; `kp`, `kd` >= 0; `tau` Nm |
| `send_pos_vel(pos, vlim)` | Position + velocity limit | After `ensure_mode(Mode.POS_VEL)` | `pos` rad; `vlim > 0` |
| `send_vel(vel)` | Pure velocity | After `ensure_mode(Mode.VEL)` | `vel` rad/s |
| `send_force_pos(pos, vlim, ratio)` | Position + torque limit | After `ensure_mode(Mode.FORCE_POS)` | `ratio` 0--1 |
| `request_feedback()` | Request one feedback frame | Before `poll_feedback_once` + `get_state` | For fresh readings |
| `get_state()` | Read cached state | After feedback | May return `None` |
| `set_can_timeout_ms(ms)` | Set CAN timeout parameter | Maintenance flow | `ms > 0` |
| `store_parameters()` | Persist registers to flash | After confirmed register writes | Irreversible per session |
| `write_register_f32(rid, value)` | Write f32 register | Optional `store_parameters` after | `rid` must be writable f32 |
| `write_register_u32(rid, value)` | Write u32 register | Optional `store_parameters` after | `rid` must be writable u32 |
| `get_register_f32(rid, timeout_ms)` | Read f32 register | Diagnostics / verification | `timeout_ms` 300--1000 |
| `get_register_u32(rid, timeout_ms)` | Read u32 register | Diagnostics / verification | `timeout_ms` 300--1000 |
| `close()` | Release motor handle | End of lifecycle | --- |

---

## 5. State and Data Structures

### 5.1 `Mode` (Enum)

```python
class Mode(IntEnum):
    MIT = 1
    POS_VEL = 2
    VEL = 3
    FORCE_POS = 4
```

### 5.2 `MotorState` (Dataclass)

Returned by `motor.get_state()`. This is a frozen dataclass.

```python
@dataclass(frozen=True)
class MotorState:
    can_id: int          # CAN ID of the motor
    arbitration_id: int  # Arbitration ID of the last feedback frame
    status_code: int     # Motor status code
    pos: float           # Position (rad)
    vel: float           # Velocity (rad/s)
    torq: float          # Torque (Nm)
    t_mos: float         # MOSFET temperature (C)
    t_rotor: float       # Rotor temperature (C)
```

Note: `MotorState` does **not** contain the current control mode. To read the current mode, query the appropriate register (e.g., Damiao `RID_CTRL_MODE = 10`).

### 5.3 `RegisterSpec` (Dataclass)

Used for Damiao register metadata lookups.

```python
@dataclass(frozen=True)
class RegisterSpec:
    rid: int          # Register ID
    variable: str     # Variable name
    description: str  # Human-readable description
    data_type: str    # "f32" or "u32"
    access: str       # Access mode (e.g., "RW")
    range_str: str    # Valid range as a string
```

---

## 6. Damiao-Specific API

### 6.1 Damiao Register Constants

Importable directly from the package:

```python
from motorbridge import (
    DAMIAO_RW_REGISTERS,
    DAMIAO_HIGH_IMPACT_RIDS,
    DAMIAO_PROTECTION_RIDS,
    get_damiao_register_spec,
    RID_CTRL_MODE,
    RID_MST_ID,
    RID_ESC_ID,
    RID_TIMEOUT,
    RID_PMAX,
    RID_VMAX,
    RID_TMAX,
    RID_KP_ASR,
    RID_KI_ASR,
    RID_KP_APR,
    RID_KI_APR,
    MODE_MIT,
    MODE_POS_VEL,
    MODE_VEL,
    MODE_FORCE_POS,
)
```

### 6.2 Damiao Register Reference Table

| RID | Variable | Type | Access | Range | Description |
|---|---|---|---|---|---|
| 0 | `UV_Value` | f32 | RW | (10.0, 3.4E38] | Under-voltage protection value |
| 1 | `KT_Value` | f32 | RW | [0.0, 3.4E38] | Torque coefficient |
| 2 | `OT_Value` | f32 | RW | [80.0, 200) | Over-temperature protection value |
| 3 | `OC_Value` | f32 | RW | (0.0, 1.0) | Over-current protection value |
| 4 | `ACC` | f32 | RW | (0.0, 3.4E38) | Acceleration |
| 5 | `DEC` | f32 | RW | [-3.4E38, 0.0) | Deceleration |
| 6 | `MAX_SPD` | f32 | RW | (0.0, 3.4E38] | Maximum speed |
| 7 | `MST_ID` | u32 | RW | [0, 0x7FF] | Feedback ID |
| 8 | `ESC_ID` | u32 | RW | [0, 0x7FF] | Receive ID |
| 9 | `TIMEOUT` | u32 | RW | [0, 2^32-1] | Timeout alarm time |
| 10 | `CTRL_MODE` | u32 | RW | [1, 4] | Control mode |
| 21 | `PMAX` | f32 | RW | (0.0, 3.4E38] | Position mapping range |
| 22 | `VMAX` | f32 | RW | (0.0, 3.4E38] | Speed mapping range |
| 23 | `TMAX` | f32 | RW | (0.0, 3.4E38] | Torque mapping range |
| 24 | `I_BW` | f32 | RW | [100.0, 10000.0] | Current loop control bandwidth |
| 25 | `KP_ASR` | f32 | RW | [0.0, 3.4E38] | Speed loop Kp |
| 26 | `KI_ASR` | f32 | RW | [0.0, 3.4E38] | Speed loop Ki |
| 27 | `KP_APR` | f32 | RW | [0.0, 3.4E38] | Position loop Kp |
| 28 | `KI_APR` | f32 | RW | [0.0, 3.4E38] | Position loop Ki |
| 29 | `OV_Value` | f32 | RW | TBD | Over-voltage protection value |
| 30 | `GREF` | f32 | RW | (0.0, 1.0] | Gear torque efficiency |
| 31 | `Deta` | f32 | RW | [1.0, 30.0] | Speed loop damping coefficient |
| 32 | `V_BW` | f32 | RW | (0.0, 500.0) | Speed loop filter bandwidth |
| 33 | `IQ_c1` | f32 | RW | [100.0, 10000.0] | Current loop enhancement coefficient |
| 34 | `VL_c1` | f32 | RW | (0.0, 10000.0] | Speed loop enhancement coefficient |
| 35 | `can_br` | u32 | RW | [0, 4] | CAN baud rate code |

### 6.3 High-Impact Register IDs

These registers have the most significant effect on motor behavior:

- `21 PMAX`, `22 VMAX`, `23 TMAX` (range limits)
- `25 KP_ASR`, `26 KI_ASR`, `27 KP_APR`, `28 KI_APR` (PID gains)
- `4 ACC`, `5 DEC`, `6 MAX_SPD`, `9 TIMEOUT` (motion limits)

### 6.4 Protection Register IDs

- `0 UV_Value` (under-voltage)
- `2 OT_Value` (over-temperature)
- `3 OC_Value` (over-current)
- `29 OV_Value` (over-voltage)

### 6.5 Damiao Parameter Read/Write (Dedicated API)

In addition to the generic `get_register_*`/`write_register_*` methods, Damiao motors have dedicated parameter access methods:

#### `damiao_get_param_f32(param_id: int, timeout_ms: int = 1000) -> float`

Reads a Damiao float parameter by index.

#### `damiao_get_param_u32(param_id: int, timeout_ms: int = 1000) -> int`

Reads a Damiao unsigned 32-bit integer parameter by index.

#### `damiao_write_param_f32(param_id: int, value: float) -> None`

Writes a Damiao float parameter by index.

#### `damiao_write_param_u32(param_id: int, value: int) -> None`

Writes a Damiao unsigned 32-bit integer parameter by index.

### 6.6 Recommended Tuning Workflow

1. Read old value: `get_register_f32(rid)` or `get_register_u32(rid)`.
2. Write new value: `write_register_f32(rid, value)` or `write_register_u32(rid, value)`.
3. Read back to verify.
4. Persist: `store_parameters()`.

Example:

```python
from motorbridge import Controller, RID_PMAX

with Controller("can0") as ctrl:
    m = ctrl.add_damiao_motor(0x01, 0x11, "4340P")
    ctrl.enable_all()

    # Read current PMAX
    old_pmax = m.get_register_f32(RID_PMAX, 500)
    print(f"Old PMAX: {old_pmax}")

    # Write new PMAX
    m.write_register_f32(RID_PMAX, 6.28)
    new_pmax = m.get_register_f32(RID_PMAX, 500)
    print(f"New PMAX: {new_pmax}")

    # Persist
    m.store_parameters()
    m.close()
```

### 6.7 Built-in Register Metadata Usage

```python
from motorbridge import (
    DAMIAO_RW_REGISTERS,
    DAMIAO_HIGH_IMPACT_RIDS,
    get_damiao_register_spec,
    RID_CTRL_MODE,
)

# Print all tunable registers with ranges
for rid in sorted(DAMIAO_RW_REGISTERS):
    spec = DAMIAO_RW_REGISTERS[rid]
    print(f"rid={rid:>2} {spec.variable:<10} type={spec.data_type} range={spec.range_str} desc={spec.description}")

# Look up a specific register
print(get_damiao_register_spec(RID_CTRL_MODE))

# List high-impact RIDs
print(DAMIAO_HIGH_IMPACT_RIDS)
```

### 6.8 Damiao Control Modes in Detail

#### MIT (Mode.MIT)

Full-parameter hybrid impedance control.

Method: `send_mit(pos, vel, kp, kd, tau)`

Parameters:
- `pos`: target position (rad)
- `vel`: target velocity (rad/s)
- `kp`: position stiffness gain (dimensionless)
- `kd`: velocity damping gain (dimensionless)
- `tau`: feedforward torque (Nm)

Typical starting values: `kp=2--10`, `kd=0.05--0.5`, `tau=0.1--1.0`.

#### POS_VEL (Mode.POS_VEL)

Position control with velocity limit.

Method: `send_pos_vel(pos, vlim)`

Parameters:
- `pos`: target position (rad)
- `vlim`: velocity limit (rad/s)

Common use case: move to a target position safely before switching to other modes.

#### VEL (Mode.VEL)

Pure velocity control.

Method: `send_vel(vel)`

Parameters:
- `vel`: target velocity (rad/s)

Common use case: continuous rotation, velocity response testing.

#### FORCE_POS (Mode.FORCE_POS)

Position control with torque/current limit.

Method: `send_force_pos(pos, vlim, ratio)`

Parameters:
- `pos`: target position (rad)
- `vlim`: velocity limit (rad/s)
- `ratio`: torque limit ratio (0--1). This is not precise constant-torque control.

Common use case: reach a target position without excessive force.

### 6.9 Damiao set_zero_position() Details

#### Command Semantics

`set_zero_position()` sets the current mechanical position as the zero-point reference. It does **not** move the motor to position 0. The underlying protocol sends the Damiao zero-set command frame (`FF FF FF FF FF FF FF FE`).

#### Internal Timing

The Python API signature is `set_zero_position() -> None` with **no `ms` parameter**. The core layer applies a fixed 20 ms settle time internally.

#### Project Rule (Important)

In this project, you **must call `disable()` before `set_zero_position()`** for Damiao motors, especially over dm-serial. This significantly reduces the risk of register read timeouts after zeroing.

Recommended sequence:

1. `disable()` (or `disable_all()`)
2. `set_zero_position()`
3. `enable()` (or `enable_all()`)
4. `ensure_mode(...)`
5. `send_*()` control

#### Error Recovery After Failure

If `set_zero_position` or subsequent operations fail:

1. `disable()` -> `clear_error()` -> `enable()` -> retry `ensure_mode()`.
2. If still failing, scan to confirm the motor is online with the expected IDs.
3. If feedback frames arrive but register reads consistently fail, consider power-cycling the device.

### 6.10 Standard Execution Flow for Damiao

1. Create controller and bind motor: `Controller(...)` + `add_damiao_motor(...)`.
2. Enable: `ctrl.enable_all()`.
3. Switch/verify control mode: `motor.ensure_mode(...)`.
4. Send commands: `send_mit` / `send_pos_vel` / `send_vel` / `send_force_pos`.
5. Read feedback as needed: `request_feedback()` + `poll_feedback_once()` + `get_state()`.
6. Disable at end: `motor.disable()` or `ctrl.disable_all()`.

### 6.11 Damiao CLI Commands

#### Scan

```bash
motorbridge-cli scan \
  --vendor damiao --channel can0 --model 4310 --start-id 0x01 --end-id 0x10
```

#### MIT Control

```bash
motorbridge-cli run \
  --vendor damiao --channel can0 --model 4340P --motor-id 0x01 --feedback-id 0x11 \
  --mode mit --pos 0 --vel 0 --kp 8 --kd 0.2 --tau 0.3 --loop 100 --dt-ms 10
```

#### Maintenance

```bash
motorbridge-cli run \
  --vendor damiao --channel can0 --model 4340P --motor-id 0x01 --feedback-id 0x11 \
  --can-timeout-ms 1000 --set-zero 0
```

---

## 7. RobStride-Specific API

### 7.1 Adding a RobStride Motor

```python
from motorbridge import Controller

with Controller("can0") as ctrl:
    motor = ctrl.add_robstride_motor(127, 0xFD, "rs-06")
```

Default feedback ID for RobStride is `0xFD` (runtime can fallback probe `0xFF`/`0xFE`).

### 7.2 robstride_ping

```python
motor.robstride_ping() -> tuple[int, int]
```

Verifies connectivity for a given `motor_id + feedback_id`. Returns `(device_id, responder_id)`.

```python
with Controller("can0") as ctrl:
    m = ctrl.add_robstride_motor(127, 0xFD, "rs-06")
    device_id, responder_id = m.robstride_ping()
    print(f"device_id={device_id}, responder_id={responder_id}")
    m.close()
```

### 7.3 robstride_set_device_id

```python
motor.robstride_set_device_id(new_device_id: int) -> None
```

Changes the device ID. Call `store_parameters()` afterward to persist.

```python
m.robstride_set_device_id(126)
m.store_parameters()
```

### 7.4 RobStride Parameter Read

RobStride provides typed parameter read methods:

| Method | Signature | Description |
|---|---|---|
| `robstride_get_param_i8` | `robstride_get_param_i8(param_id: int, timeout_ms: int = 1000) -> int` | Read signed 8-bit parameter |
| `robstride_get_param_u8` | `robstride_get_param_u8(param_id: int, timeout_ms: int = 1000) -> int` | Read unsigned 8-bit parameter |
| `robstride_get_param_u16` | `robstride_get_param_u16(param_id: int, timeout_ms: int = 1000) -> int` | Read unsigned 16-bit parameter |
| `robstride_get_param_u32` | `robstride_get_param_u32(param_id: int, timeout_ms: int = 1000) -> int` | Read unsigned 32-bit parameter |
| `robstride_get_param_f32` | `robstride_get_param_f32(param_id: int, timeout_ms: int = 1000) -> float` | Read float parameter |

Common RobStride parameters:
- `0x7005` `run_mode`
- `0x7016` `loc_ref`
- `0x7017` `limit_spd`
- `0x700A` `spd_ref`
- `0x7019` `mechPos`
- `0x701B` `mechVel`

Example:

```python
with Controller("can0") as ctrl:
    m = ctrl.add_robstride_motor(127, 0xFD, "rs-06")
    pos = m.robstride_get_param_f32(0x7019, 200)
    print(f"mechPos = {pos}")
    m.close()
```

### 7.5 RobStride Parameter Write

| Method | Signature | Description |
|---|---|---|
| `robstride_write_param_i8` | `robstride_write_param_i8(param_id: int, value: int) -> None` | Write signed 8-bit parameter |
| `robstride_write_param_u8` | `robstride_write_param_u8(param_id: int, value: int) -> None` | Write unsigned 8-bit parameter |
| `robstride_write_param_u16` | `robstride_write_param_u16(param_id: int, value: int) -> None` | Write unsigned 16-bit parameter |
| `robstride_write_param_u32` | `robstride_write_param_u32(param_id: int, value: int) -> None` | Write unsigned 32-bit parameter |
| `robstride_write_param_f32` | `robstride_write_param_f32(param_id: int, value: float) -> None` | Write float parameter |

Example:

```python
with Controller("can0") as ctrl:
    m = ctrl.add_robstride_motor(127, 0xFD, "rs-06")
    m.robstride_write_param_f32(0x700A, 0.3)
    # Read back to verify
    print(m.robstride_get_param_f32(0x700A, 200))
    m.close()
```

### 7.6 RobStride Mode Mapping

| Unified Mode | RobStride Native |
|---|---|
| `Mode.MIT` | Native MIT impedance control frame (`pos`/`vel`/`kp`/`kd`/`tau` all active) |
| `Mode.POS_VEL` | Maps to native Position: `run_mode=1`, `loc_ref(0x7016)`, `limit_spd(0x7017)` |
| `Mode.VEL` | Native Velocity: `run_mode=2`, `spd_ref(0x700A)` |
| `Mode.FORCE_POS` | Unsupported |

Note: Torque/current control for RobStride is parameter-level only (`robstride_write_param_*`), not a dedicated unified mode.

### 7.7 RobStride Scan (Python SDK)

```python
from motorbridge import Controller

found = []
with Controller("can0") as ctrl:
    for mid in range(120, 131):
        try:
            m = ctrl.add_robstride_motor(mid, 0xFD, "rs-06")
            try:
                print(mid, m.robstride_ping())
                found.append(mid)
            finally:
                m.close()
        except Exception:
            pass
print("found:", found)
```

### 7.8 RobStride Zero-Set Sequence

```python
from motorbridge import Controller

with Controller("can0") as ctrl:
    m = ctrl.add_robstride_motor(127, 0xFD, "rs-06")
    m.disable()
    m.set_zero_position()
    m.store_parameters()
    m.close()
```

Verify after zero-set:

```bash
motorbridge-cli robstride-read-param \
  --channel can0 --model rs-06 --motor-id 127 --feedback-id 0xFD \
  --param-id 0x7019 --type f32 --timeout-ms 200
```

---

## 8. Error Handling

### 8.1 Exception Hierarchy

```
MotorBridgeError (RuntimeError)
  +-- AbiLoadError
  +-- CallError
```

### 8.2 Exception Classes

#### `MotorBridgeError`

Base exception for all `motorbridge` Python SDK errors. Inherits from `RuntimeError`.

#### `AbiLoadError`

Raised when the `libmotor_abi` shared library cannot be found or loaded. The error message lists all searched paths.

#### `CallError`

Raised when an ABI call returns a non-zero status code. The error message includes the operation name and the underlying error text from the ABI layer.

### 8.3 Common Error Patterns

```python
from motorbridge import Controller, MotorBridgeError, AbiLoadError, CallError

try:
    with Controller("can0") as ctrl:
        motor = ctrl.add_damiao_motor(0x01, 0x11, "4340P")
        ctrl.enable_all()
        motor.ensure_mode(Mode.MIT, 1000)
except AbiLoadError as e:
    print(f"Library load failed: {e}")
except CallError as e:
    print(f"API call failed: {e}")
except MotorBridgeError as e:
    print(f"MotorBridge error: {e}")
```

### 8.4 Error Recovery for Damiao

If a motor enters an error state:

1. `disable()` -> `clear_error()` -> `enable()` -> retry `ensure_mode()`.
2. If still failing, scan to confirm the motor is online with the expected IDs.
3. If feedback frames arrive but register reads consistently fail, consider power-cycling the device.

---

## 9. Unified Mode Mapping

| Unified Mode | Damiao | RobStride | Hexfellow | MyActuator | HighTorque |
|---|---|---|---|---|---|
| `Mode.MIT` | Native MIT | Native MIT | Native MIT (mode 5) | Unsupported | Maps to native pos+vel+tqe |
| `Mode.POS_VEL` | Native POS_VEL | Native Position (`run_mode=1` + `limit_spd(0x7017)` + `loc_ref(0x7016)`) | Native POS_VEL (mode 1) | Position setpoint flow | Maps to native pos+vel+tqe |
| `Mode.VEL` | Native VEL | Native Velocity | Unsupported | Native velocity setpoint | Native velocity command |
| `Mode.FORCE_POS` | Native FORCE_POS | Unsupported | Unsupported | Unsupported | Maps to native pos+vel+tqe |

---

## 10. Example Programs

The `motorbridge` package ships with example scripts in `bindings/python/examples/`.

### Damiao Examples

| Script | Description |
|---|---|
| `python_wrapper_demo.py` | Minimal MIT closed-loop demo |
| `full_modes_demo.py` | Unified entry for all four Damiao modes |
| `scan_ids_demo.py` | Damiao ID scanning |
| `pos_ctrl_demo.py` | Single position target |
| `multi_motor_ctrl_demo.py` | Multi-motor control with per-step SDK timing |
| `mit_pos_switch_demo.py` | Multi-motor mode switching test (MIT -> POS_VEL) |
| `pos_repl_demo.py` | Interactive position control |
| `pid_register_tune_demo.py` | Register tuning |
| `damiao_maintenance_demo.py` | Maintenance: clear_error, set_zero, timeout, feedback |
| `damiao_register_rw_demo.py` | Register f32/u32 read/write + store |
| `damiao_dm_serial_demo.py` | Damiao serial bridge transport demo |
| `dm_serial_mode_switch_200_demo.py` | dm-serial continuous 4-mode switching (configurable loops per mode) |
| `dm_serial_status_like_cli_demo.py` | dm-serial query current mode + key registers + live state |
| `dm_serial_leader_monitor_demo.py` | dm-serial leader monitor (full state reporting + startup enable-all) |

### RobStride Examples

| Script | Description |
|---|---|
| `robstride_wrapper_demo.py` | Ping, velocity, position, MIT demos |

### Running Examples (Local Build)

```bash
cd motorbridge
cargo build -p motor_abi --release
export PYTHONPATH=bindings/python/src
export LD_LIBRARY_PATH=$PWD/target/release:${LD_LIBRARY_PATH}

# Damiao MIT demo
python3 bindings/python/examples/python_wrapper_demo.py \
  --channel can0 --model 4340P --motor-id 0x01 --feedback-id 0x11 \
  --pos 0 --vel 0 --kp 20 --kd 1 --tau 0 --loop 20 --dt-ms 20

# RobStride ping demo
python3 bindings/python/examples/robstride_wrapper_demo.py \
  --channel can0 --model rs-06 --motor-id 127 --feedback-id 0xFD --mode ping

# RobStride velocity demo
python3 bindings/python/examples/robstride_wrapper_demo.py \
  --channel can0 --model rs-06 --motor-id 127 --feedback-id 0xFD \
  --mode vel --vel 0.3 --loop 40 --dt-ms 50
```

---

## Full Damiao Tuning Table

For the complete Damiao operation/tuning table and additional command examples, see:
- [`motor_cli/DAMIAO_API.md`](../../../motorbridge/motor_cli/DAMIAO_API.md)
- [`motor_cli/DAMIAO_API.zh-CN.md`](../../../motorbridge/motor_cli/DAMIAO_API.zh-CN.md)
