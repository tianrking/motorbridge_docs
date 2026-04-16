# Damiao CLI 子命令

本页是 `motorbridge` 当前 Damiao 可调控制/配置参数的实用总表。

> English version: [damiao.md](damiao.md)

**相关文档：** [CLI 参考](reference.zh-CN.md) | [snippets/channel-compat.md](../snippets/channel-compat.md)

## 通道兼容详情

> **通道兼容详情：** 参见 [snippets/channel-compat.md](../snippets/channel-compat.md)

## 支持模式

| 模式 | 说明 |
|---|---|
| `scan` | 在 ID 范围内探测电机 |
| `enable` | 使能电机（退出阻尼） |
| `disable` | 失能电机（进入阻尼） |
| `mit` | MIT 阻抗控制（`pos/vel/kp/kd/tau`） |
| `pos-vel` | 位置速度控制（`pos/vlim`） |
| `vel` | 速度控制（`vel`） |
| `force-pos` | 力位混合控制（`pos/vlim/ratio`） |

## 1）通用设备参数

| 参数 | 含义 | 常用值 |
|---|---|---|
| `channel` | CAN 接口名 | `can0` |
| `model` | 电机型号字符串（`4310`、`4340P` 等） | 与实物一致 |
| `motor-id` | 命令 ID（`ESC_ID`） | 如 `0x01` |
| `feedback-id` | 反馈 ID（`MST_ID`） | 如 `0x11` |
| `loop` | 发送循环次数 | `100` |
| `dt-ms` | 每次发送间隔（毫秒） | `20` |

## 2）扫描（scan）

在指定 CAN ID 范围内扫描 Damiao 电机。

### 参数

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `--start-id` | u16 | `1` | 扫描起始 ID（1..255） |
| `--end-id` | u16 | `255` | 扫描结束 ID（1..255） |

### 扫描行为细节

- 扫描逻辑本质上是"型号无关"的：内部会遍历内置 model-hint 列表。
- 每个候选 ID 会尝试多个 feedback-hint：推断值（`id+0x10`）、用户给定 `--feedback-id`、`0x11`、`0x17`。
- 优先用寄存器（RID 21/22/23）检测，失败再走反馈回退检测。

### 示例

```bash
motor_cli \
  --vendor damiao --channel can0 --mode scan --start-id 1 --end-id 16
# [STD-CAN]
```

## 3）使能 / 失能（enable / disable）

使能或失能（阻尼）电机。

```bash
# 使能
motor_cli \
  --vendor damiao --channel can0 --model 4310 --motor-id 0x04 --feedback-id 0x14 \
  --mode enable --loop 1

# 失能
motor_cli \
  --vendor damiao --channel can0 --model 4310 --motor-id 0x04 --feedback-id 0x14 \
  --mode disable --loop 1
```

`enable`/`disable` 无额外控制参数。

## 4）MIT 模式

参数：

- `pos`：目标位置
- `vel`：目标速度
- `kp`：位置刚度增益
- `kd`：速度阻尼增益
- `tau`：前馈力矩

### 参数

| 参数 | 类型 | 默认值 |
|---|---|---|
| `--pos` | f32 | `0` |
| `--vel` | f32 | `0` |
| `--kp` | f32 | `2` |
| `--kd` | f32 | `1` |
| `--tau` | f32 | `0` |

### 示例

```bash
# 标准 CAN 下 MIT 控制
motor_cli \
  --vendor damiao --channel can0 --model 4310 --motor-id 0x04 --feedback-id 0x14 \
  --mode mit --pos 1.57 --vel 2.0 --kp 35 --kd 1.2 --tau 0.3 --loop 120 --dt-ms 20
# [STD-CAN]

# 通过 Damiao 串口桥执行 MIT
motor_cli \
  --vendor damiao --transport dm-serial --serial-port /dev/ttyACM1 --serial-baud 921600 \
  --model 4310 --motor-id 0x04 --feedback-id 0x14 \
  --mode mit --verify-model 0 --ensure-mode 0 \
  --pos 1.0 --vel 0 --kp 2 --kd 1 --tau 0 --loop 80 --dt-ms 20
# [DM-SERIAL]

# 通过独立 CAN-FD 链路执行 MIT
motor_cli \
  --vendor damiao --transport socketcanfd --channel can0 \
  --model 4310 --motor-id 0x04 --feedback-id 0x14 \
  --mode mit --verify-model 0 --ensure-mode 0 \
  --pos 0.5 --vel 0 --kp 20 --kd 1 --tau 0 --loop 80 --dt-ms 20
# [CAN-FD]
```

## 5）POS_VEL 模式

参数：

- `pos`：目标位置
- `vlim`：速度限制

### 参数

| 参数 | 类型 | 默认值 |
|---|---|---|
| `--pos` | f32 | `0` |
| `--vlim` | f32 | `1.0` |

### 示例

```bash
motor_cli \
  --vendor damiao --channel can0 --model 4340P --motor-id 0x01 --feedback-id 0x11 \
  --mode pos-vel --pos 3.10 --vlim 1.50 --loop 300 --dt-ms 20
# [STD-CAN]
```

## 6）VEL 模式

参数：

- `vel`：目标速度

### 参数

| 参数 | 类型 | 默认值 |
|---|---|---|
| `--vel` | f32 | `0` |

### 示例

```bash
motor_cli \
  --vendor damiao --channel can0 --model 4340P --motor-id 0x01 --feedback-id 0x11 \
  --mode vel --vel 0.5 --loop 100 --dt-ms 20
# [STD-CAN]
```

## 7）FORCE_POS 模式

参数：

- `pos`：目标位置
- `vlim`：速度限制
- `ratio`：力矩限制比例

### 参数

| 参数 | 类型 | 默认值 |
|---|---|---|
| `--pos` | f32 | `0` |
| `--vlim` | f32 | `1.0` |
| `--ratio` | f32 | `0.1` |

### 示例

```bash
motor_cli \
  --vendor damiao --channel can0 --model 4340P --motor-id 0x01 --feedback-id 0x11 \
  --mode force-pos --pos 0.8 --vlim 2.0 --ratio 0.3 --loop 100 --dt-ms 20
# [STD-CAN]
```

## 8）模式寄存器（`CTRL_MODE`）

寄存器：

- `rid=10`（`CTRL_MODE`），取值：
  - `1=MIT`
  - `2=POS_VEL`
  - `3=VEL`
  - `4=FORCE_POS`

## 9）Damiao 专用参数

| 参数 | 类型 | 默认值 | 作用范围 | 说明 |
|---|---|---|---|---|
| `--verify-model` | `0/1` | `1` | 非 scan | 校验 PMAX/VMAX/TMAX 与 `--model` 一致 |
| `--verify-timeout-ms` | u64 | `500` | 非 scan | 型号握手读取超时 |
| `--verify-tol` | f32 | `0.2` | 非 scan | 限值匹配容差 |
| `--set-motor-id` | u16 可选 | 无 | 改 ID 流程 | 写 ESC_ID（RID 8） |
| `--set-feedback-id` | u16 可选 | 无 | 改 ID 流程 | 写 MST_ID（RID 7） |
| `--store` | `0/1` | `1` | 改 ID 流程 | 是否保存参数 |
| `--verify-id` | `0/1` | `1` | 改 ID 流程 | 是否回读 RID7/RID8 校验 |

## 10）强影响寄存器（优先）

这些参数对控制效果影响很大，调参要谨慎。

| RID | 名称 | 类型 | 含义 |
|---|---|---|---|
| `21` | `PMAX` | `f32` | 位置映射范围 |
| `22` | `VMAX` | `f32` | 速度映射范围 |
| `23` | `TMAX` | `f32` | 力矩映射范围 |
| `25` | `KP_ASR` | `f32` | 速度环 Kp |
| `26` | `KI_ASR` | `f32` | 速度环 Ki |
| `27` | `KP_APR` | `f32` | 位置环 Kp |
| `28` | `KI_APR` | `f32` | 位置环 Ki |
| `4` | `ACC` | `f32` | 加速度 |
| `5` | `DEC` | `f32` | 减速度 |
| `6` | `MAX_SPD` | `f32` | 最大速度 |
| `9` | `TIMEOUT` | `u32` | 通信超时寄存器 |

## 11）保护相关寄存器

| RID | 名称 | 类型 | 含义 |
|---|---|---|---|
| `0` | `UV_Value` | `f32` | 欠压阈值 |
| `2` | `OT_Value` | `f32` | 过温阈值 |
| `3` | `OC_Value` | `f32` | 过流阈值 |
| `29` | `OV_Value` | `f32` | 过压阈值 |

## 12）参数读写方法

推荐流程：

1. `get_register` 读取旧值
2. `write_register` 写入新值
3. 再次 `get_register` 回读确认
4. `store_parameters` 持久化

### Python SDK API 示例

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

### WS 网关命令示例

```json
{"op":"get_register_f32","rid":22,"timeout_ms":1000}
{"op":"write_register_f32","rid":22,"value":25.0}
{"op":"store_parameters"}
```

## 13）ID 与标定参数

- `rid=8`（`ESC_ID`）：命令 ID
- `rid=7`（`MST_ID`）：反馈 ID

### 改 ID 示例

```bash
# 改 ID + 保存 + 校验
motor_cli \
  --vendor damiao --channel can0 --model 4310 --motor-id 0x01 --feedback-id 0x11 \
  --set-motor-id 0x04 --set-feedback-id 0x14 --store 1 --verify-id 1
```

### 工具入口

- Rust CLI：`motor_cli --vendor damiao --mode scan ...` 以及 `--set-motor-id/--set-feedback-id --store --verify-id`
- Python：`motorbridge-cli scan/id-set/id-dump`
- WS：`scan`、`set_id`、`verify`

## 14）安全建议

- 每次只调一组参数。
- 小步进调整，每一步都回读确认。
- 保持机械安全和急停预案。
- 对保护阈值（`0/2/3/29`）不要盲目激进调整。

## 15）夹爪电机校准

Damiao 电机使用单圈编码器（位置范围约 +/-PMAX rad），**断电后零点不保持**。当电机用于夹爪时，每次上电需先校准零位：

1. MIT 模式低力矩推向机械限位，`kp` 低、`tau=0`，碰到即软停。
2. 等反馈 `vel ~ 0` 且 `pos` 稳定后，执行 `--mode set-zero`。
3. 之后闭合用 MIT（力控安全），张开用 pos-vel（快速精确）。

详细命令见操作手册第 9 章。
