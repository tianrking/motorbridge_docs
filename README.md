# motorbridge Documentation

> Unified CAN motor control — Rust core, Python/C++ bindings, CLI, integrations.

## Quick Links

| Section | English | 中文 |
|---|---|---|
| Quick Start | [getting-started/quickstart.md](getting-started/quickstart.md) | [getting-started/quickstart.zh-CN.md](getting-started/quickstart.zh-CN.md) |
| Courses (10 lessons) | [getting-started/courses/](getting-started/courses/) | [getting-started/courses/overview.zh-CN.md](getting-started/courses/overview.zh-CN.md) |

## Sections

### Getting Started

| File | Description |
|---|---|
| [getting-started/quickstart.md](getting-started/quickstart.md) | Install, build, first scan and control |
| [getting-started/quickstart.zh-CN.md](getting-started/quickstart.zh-CN.md) | 快速开始 |
| [getting-started/courses/overview.zh-CN.md](getting-started/courses/overview.zh-CN.md) | 10 课总览与绑定接口速查 |
| [getting-started/courses/00-enable-and-status.md](getting-started/courses/00-enable-and-status.md) | Course 00 — Enable motor and read status |
| [getting-started/courses/01-scan.md](getting-started/courses/01-scan.md) | Course 01 — Scan for motors |
| [getting-started/courses/02-register-rw.md](getting-started/courses/02-register-rw.md) | Course 02 — Register read/write |
| [getting-started/courses/03-mode-switch-method.md](getting-started/courses/03-mode-switch-method.md) | Course 03 — Mode switch methods |
| [getting-started/courses/04-mode-mit.md](getting-started/courses/04-mode-mit.md) | Course 04 — MIT mode |
| [getting-started/courses/05-mode-pos-vel.md](getting-started/courses/05-mode-pos-vel.md) | Course 05 — POS_VEL mode |
| [getting-started/courses/06-mode-vel.md](getting-started/courses/06-mode-vel.md) | Course 06 — VEL mode |
| [getting-started/courses/07-mode-force-pos.md](getting-started/courses/07-mode-force-pos.md) | Course 07 — FORCE_POS mode |
| [getting-started/courses/08-mode-mixed-switch.md](getting-started/courses/08-mode-mixed-switch.md) | Course 08 — Mixed mode switching |
| [getting-started/courses/09-multi-motor.md](getting-started/courses/09-multi-motor.md) | Course 09 — Multi-motor single query |

### Architecture

| File | Description |
|---|---|
| [architecture/overview.md](architecture/overview.md) | System architecture, crate layout, lifecycle |
| [architecture/overview.zh-CN.md](architecture/overview.zh-CN.md) | 系统架构概览 |
| [architecture/abi.md](architecture/abi.md) | C ABI design and FFI layer |
| [architecture/abi.zh-CN.md](architecture/abi.zh-CN.md) | C ABI 设计与 FFI 层 |
| [architecture/extending.md](architecture/extending.md) | Adding a new vendor driver |
| [architecture/extending.zh-CN.md](architecture/extending.zh-CN.md) | 新增厂商驱动 |

### SDK — Python

| File | Description |
|---|---|
| [sdk/python/reference.md](sdk/python/reference.md) | Python SDK API reference |
| [sdk/python/reference.zh-CN.md](sdk/python/reference.zh-CN.md) | Python SDK API 参考 |
| [sdk/python/examples.md](sdk/python/examples.md) | Python examples index |
| [sdk/python/examples.zh-CN.md](sdk/python/examples.zh-CN.md) | Python 示例索引 |
| [sdk/python/robstride.zh-CN.md](sdk/python/robstride.zh-CN.md) | RobStride Python 绑定 |

### SDK — C++

| File | Description |
|---|---|
| [sdk/cpp/reference.md](sdk/cpp/reference.md) | C++ binding API reference |
| [sdk/cpp/reference.zh-CN.md](sdk/cpp/reference.zh-CN.md) | C++ 绑定 API 参考 |
| [sdk/cpp/examples.md](sdk/cpp/examples.md) | C++ examples |
| [sdk/cpp/examples.zh-CN.md](sdk/cpp/examples.zh-CN.md) | C++ 示例 |

### SDK — C

| File | Description |
|---|---|
| [sdk/c/examples.md](sdk/c/examples.md) | C ABI examples |
| [sdk/c/examples.zh-CN.md](sdk/c/examples.zh-CN.md) | C ABI 示例 |

### CLI

| File | Description |
|---|---|
| [cli/reference.md](cli/reference.md) | CLI general reference |
| [cli/reference.zh-CN.md](cli/reference.zh-CN.md) | CLI 通用参考 |
| [cli/damiao.md](cli/damiao.md) | Damiao CLI sub-commands |
| [cli/damiao.zh-CN.md](cli/damiao.zh-CN.md) | Damiao CLI 子命令 |
| [cli/myactuator.md](cli/myactuator.md) | MyActuator CLI sub-commands |
| [cli/myactuator.zh-CN.md](cli/myactuator.zh-CN.md) | MyActuator CLI 子命令 |
| [cli/robstride.md](cli/robstride.md) | RobStride CLI sub-commands |
| [cli/robstride.zh-CN.md](cli/robstride.zh-CN.md) | RobStride CLI 子命令 |

### Devices

| File | Description |
|---|---|
| [devices/supported-devices.md](devices/supported-devices.md) | Supported motors by vendor |
| [devices/supported-devices.zh-CN.md](devices/supported-devices.zh-CN.md) | 支持的电机型号 |
| [devices/hightorque-protocol.zh-CN.md](devices/hightorque-protocol.zh-CN.md) | HighTorque 协议分析 |
| [devices/operation-manual.zh-CN.md](devices/operation-manual.zh-CN.md) | 操作手册 |

### Guides

| File | Description |
|---|---|
| [guides/can-debugging.md](guides/can-debugging.md) | CAN bus debugging (Linux + Windows) |
| [guides/can-debugging.zh-CN.md](guides/can-debugging.zh-CN.md) | CAN 总线排障 |
| [guides/testing.md](guides/testing.md) | Testing guide |
| [guides/testing.zh-CN.md](guides/testing.zh-CN.md) | 测试指南 |
| [guides/distribution.md](guides/distribution.md) | Distribution channels (CI) |
| [guides/distribution.zh-CN.md](guides/distribution.zh-CN.md) | 分发渠道 |
| [guides/windows.md](guides/windows.md) | Windows SDK usage |
| [guides/windows.zh-CN.md](guides/windows.zh-CN.md) | Windows SDK 使用 |

### Integrations

| File | Description |
|---|---|
| [integrations/ros2-bridge.md](integrations/ros2-bridge.md) | ROS2 bridge setup |
| [integrations/ros2-bridge.zh-CN.md](integrations/ros2-bridge.zh-CN.md) | ROS2 桥接 |

### Tools

| File | Description |
|---|---|
| [tools/factory-calibration.md](tools/factory-calibration.md) | Factory calibration UI |
| [tools/factory-calibration.zh-CN.md](tools/factory-calibration.zh-CN.md) | 工厂标定 UI |
| [tools/reliability.md](tools/reliability.md) | Reliability runner |
| [tools/reliability.zh-CN.md](tools/reliability.zh-CN.md) | 可靠性测试 |

### Examples

| File | Description |
|---|---|
| [examples/damiao-all-in-one.md](examples/damiao-all-in-one.md) | Damiao all-in-one demo |
| [examples/web-hmi.md](examples/web-hmi.md) | Web HMI demo |
| [examples/web-hmi.zh-CN.md](examples/web-hmi.zh-CN.md) | Web HMI 示例 |
| [examples/http-quad-control.zh-CN.md](examples/http-quad-control.zh-CN.md) | 四路滑杆控制示例 |

### Contributing

| File | Description |
|---|---|
| [contributing/contributing.md](contributing/contributing.md) | Contribution guidelines |
| [contributing/security.md](contributing/security.md) | Security policy |
| [contributing/code-of-conduct.md](contributing/code-of-conduct.md) | Code of conduct |

### Shared Snippets

| File | Description |
|---|---|
| [snippets/channel-compat.md](snippets/channel-compat.md) | Channel compatibility reference (PCAN, slcan, CAN-FD, serial) |

## Vendor Support Matrix

| Vendor | Models | MIT | POS_VEL | VEL | FORCE_POS | Serial Bridge |
|---|---|---|---|---|---|---|
| Damiao | 4310, 4340P | Yes | Yes | Yes | Yes | Yes |
| MyActuator | X8, X6 | Yes | Yes | Yes | — | — |
| RobStride | rs-00 | Yes | — | Yes | — | — |
| HighTorque | (protocol analysis) | — | — | — | — | — |
| Hexfellow | (CAN-FD) | Yes | Yes | — | — | — |

## Conventions

- **Channel format**: `can0@1000000` (Linux SocketCAN), `can0@1000000` (Windows PCAN), `slcan0@1000000` (slcan)
- **Bilingual**: Each doc has EN (`*.md`) and ZH (`*.zh-CN.md`) versions
- **Channel compatibility**: All channel-related notes link to [snippets/channel-compat.md](snippets/channel-compat.md)
