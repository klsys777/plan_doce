# 参数表 RW 配置参数实施方案

本文档整合了“参数表管理器增加第一优先级、第二优先级可写配置参数”的方案，。

## 背景

原参数表 `doc/params_20260511_174142.json` 主要覆盖状态数据读取：

- 读取路径：`ProtocolParser::buildGetStatus()` -> `CAN_CMD_GET_FINGER_STATUS_DATA`
- 参数定位：依赖 `data_offset` / `data_count` 解码状态 payload
- 权限：原状态项为 `permission: "RO"` / `readonly: true`

协议 1~17 是 CAN 电机命令，不是参数表 JSON 的 `index`。其中 1~6 属于动作类命令（使能、失能、校准、复位、回零等），不适合做成普通参数；8~17 属于驱动配置类命令，适合通过参数表读写。

## 原实施方案

### 纳入参数表的命令

第一优先级：

| 命令 | 参数 | 类型 | 缩放 |
| --- | --- | --- | --- |
| `CAN_CMD_SET_CAN_TIMOUT` | CAN 超时保护时间 | `uint16` | `1` |
| `CAN_CMD_SET_LIMIT_CURRENT` | 电流限制 | `float` | `A * 10` |
| `CAN_CMD_SET_LIMIT_VEL` | 速度限制 | `float` | `turn/s * 10` |
| `CAN_CMD_SET_LIMIT_PPOS` | 3 关节正位置限位 | `float[3]` | `turn * 10` |
| `CAN_CMD_SET_LIMIT_NPOS` | 3 关节负位置限位 | `float[3]` | `turn * 10` |
| `CAN_CMD_SET_CONTROL_MODE` | 控制模式 | `uint8` | `1` |

第二优先级：

| 命令 | 参数 | 类型 | 缩放 |
| --- | --- | --- | --- |
| `CAN_CMD_SET_BANDWIDTH` | 电流环带宽 | `uint16` | `1` |
| `CAN_CMD_SET_MOS_TEMPERATURE` | MOS 过温阈值 | `float` | `℃ * 10` |
| `CAN_CMD_SET_OVER_VOLTAGE` | 过压保护 | `float` | `V * 10` |
| `CAN_CMD_SET_UNDER_VOLTAGE` | 欠压保护 | `float` | `V * 10` |

协议 8~17 实际是 10 类配置项。按 5 个手指展开后，共新增 50 个 RW 配置参数。

### 不纳入参数表的命令

以下命令保留现有按钮或专用功能入口，不作为普通参数：

- `CAN_CMD_MOTOR_DISABLE`
- `CAN_CMD_MOTOR_ENABLE`
- `CAN_CMD_MOTOR_CALIBRATION`
- `CAN_CMD_DRIVE_RESET`
- `CAN_CMD_SET_HOME`
- `CAN_CMD_SET_ZERO`
- `CAN_CMD_SET_ID`

原因是这些命令属于动作或设备身份配置，误触发风险高，不适合和普通参数读写混在一起。

## 实际实现

### 1. 参数模型扩展

文件：`include/paramtablewindow.h`

`ParamItem` 新增驱动配置写入元数据：

```cpp
QString txCmd;
int fingerSlot = -1;
double scale = 1.0;
int txPayloadOffset = 0;
```

含义：

- `txCmd`：写入/读回使用的电机命令名，例如 `CAN_CMD_SET_LIMIT_CURRENT`
- `fingerSlot`：目标手指，范围 `0~4`
- `scale`：UI 值到协议值的缩放比例
- `txPayloadOffset`：为后续协议扩展预留，当前配置项为 `0`

后续整理中没有保留 `tx_pc_cmd` 字段。PC 命令类型由 `ProtocolParser::buildSetDriveConfig()` 根据 `MotCmd` 统一路由，避免 JSON 中出现无效路由字段。

### 2. 参数 JSON 合并

文件：`doc/params_20260511_174142.json`

最终没有新增独立的 `config_params_rw.json` 文件，而是将 50 个 RW 配置项合并进现有参数表。

当前结构：

- 总参数数：162
- RW 配置参数数：50
- 每个手指 10 个 RW 配置参数
- RW 参数分别放在对应的 `Finger_0` ~ `Finger_4` 分组下

示例：

```json
{
    "name": "F0_limit_cur",
    "group": "Finger_0",
    "type": "float",
    "permission": "RW",
    "readonly": false,
    "value": 0,
    "unit": "A",
    "min": 0,
    "max": 1000,
    "description": "拇指电流限制",
    "rx_source_cmd": "CAN_CMD_SET_LIMIT_CURRENT",
    "data_offset": -1,
    "data_count": 1,
    "tx_cmd": "CAN_CMD_SET_LIMIT_CURRENT",
    "finger_slot": 0,
    "scale": 10,
    "tx_payload_offset": 0
}
```

说明：

- RO 状态参数继续使用 `data_offset >= 0`
- RW 配置参数使用 `tx_cmd` + `finger_slot`
- RW 配置参数的 `data_offset` 固定为 `-1`
- 读取和写入都通过 `tx_cmd` 分发，而不是复用 `CAN_CMD_WRITE_PARA + data_offset`

### 3. 参数表 UI 支持

文件：`src/paramtablewindow.cpp`

主要变化：

- 导入 JSON 时解析 `tx_cmd`、`finger_slot`、`scale`、`tx_payload_offset`
- 导出 JSON 时保留上述字段
- 表格列：`Index, Name, Group, Type, Access, Value, Unit, Min, Max, Cmd, Offset, Count, Description`（13 列）
- `Cmd` 为 UI 合并列：驱动配置 RW 显示 `tx_cmd`；RO 状态项显示 `rx_source_cmd`；JSON 仍保留 `rx_source_cmd` / `tx_cmd` 两个字段
- 驱动配置 RW 行（`tx_cmd` 非空）仅 `Value` 可编辑，协议元数据列锁定
- `Description` 显示短中文名；完整枚举说明通过单元格 tooltip 展示
- `CAN_CMD_SET_CONTROL_MODE` 支持中文/枚举式输入
- 多值参数（例如 `limit_ppos` / `limit_npos`）支持数组值读写

### 4. 主窗口读写分发

文件：`include/mainwindow.h`、`src/mainwindow.cpp`

新增核心函数：

- `buildParamReadCommand(const ParamItem&, QByteArray&)`
- `buildParamWriteCommand(const ParamItem&, QByteArray&)`
- `applyDriveConfigResponse(FingerIndex, MotCmd, const QByteArray&)`

分发规则：

| 参数类型 | 识别方式 | 读取路径 | 写入路径 |
| --- | --- | --- | --- |
| RO 状态参数 | 无 `tx_cmd`，有 `data_offset` | `buildGetStatus()` | 不写 |
| RW 配置参数 | 有 `tx_cmd`，有 `finger_slot` | `buildReadDriveConfig()` | `buildSetDriveConfig()` |
| 旧普通写参 | 非驱动配置，`data_offset >= 0` | 旧状态读取 | `cmdBuildWritePara(..., CAN_CMD_WRITE_PARA, data_offset + value)` |

关键点：

- `buildParamReadCommand()` 对驱动配置项发送空 payload 的配置命令，用作读回
- `buildParamWriteCommand()` 对驱动配置项按 `scale` 编码 payload
- 写入成功后保留 `buildSaveParam()` 入队逻辑，用于 EEPROM 持久化
- 驱动配置响应通过 `onRxSetMode()` 回填参数表

### 5. 协议层接口

文件：`include/protocolparser.h`、`src/protocolparser.cpp`

新增接口：

```cpp
QByteArray buildSetDriveConfig(MotCmd motorCmd,
                               FingerIndex fingerIndex,
                               const QByteArray &figerData = QByteArray());

QByteArray buildReadDriveConfig(MotCmd motorCmd, FingerIndex fingerIndex);
```

路由规则：

- `CAN_CMD_SET_CONTROL_MODE` -> `cmdBuildSetControl()` -> `PC_CMD_MOTOR_CONTROL`
- 其他 9 个配置命令 -> `cmdBuildSetMode()` -> `PC_CMD_SET_MODE`
- 读配置使用同一命令但 payload 为空

`CAN_CMD_SET_CONTROL_MODE` 目前存在两条可用路径：

| 路径 | 入口 | 作用 |
| --- | --- | --- |
| 原控制面板 | `buildSetCtrlMode()` | 全手控制模式设置 |
| 参数表 | `buildSetDriveConfig(CAN_CMD_SET_CONTROL_MODE, fingerIndex, payload)` | 单指控制模式读写 |

二者行为不同，保留并行实现。

### 6. 串口线程响应帧数量

文件：`src/serialcommthread.cpp`

`expectedFrameCount` 增加驱动配置命令：

- `CAN_CMD_SET_CAN_TIMOUT`
- `CAN_CMD_SET_BANDWIDTH`
- `CAN_CMD_SET_LIMIT_CURRENT`
- `CAN_CMD_SET_LIMIT_VEL`
- `CAN_CMD_SET_LIMIT_PPOS`
- `CAN_CMD_SET_LIMIT_NPOS`
- `CAN_CMD_SET_MOS_TEMPERATURE`
- `CAN_CMD_SET_OVER_VOLTAGE`
- `CAN_CMD_SET_UNDER_VOLTAGE`
- `CAN_CMD_SET_CONTROL_MODE`

规则：

- 目标手指为 `ALL_Finger` 时等待 5 帧
- 目标为单指时等待 1 帧

这是对既有 `SET_HOME` / `SET_ZERO` / `SET_CONTROL_MODE` 分支的扩展，不是新建第二套计数逻辑。


## 当前可用范围

### 已有独立 UI 的命令

目前只有 `CAN_CMD_SET_CONTROL_MODE` 已有主控制面板入口，同时也可通过参数表单指读写。

### 仅保留参数表接口的命令

以下 9 个命令没有独立 UI 控件，但已具备参数表泛型读写接口：

- `CAN_CMD_SET_CAN_TIMOUT`
- `CAN_CMD_SET_BANDWIDTH`
- `CAN_CMD_SET_LIMIT_CURRENT`
- `CAN_CMD_SET_LIMIT_VEL`
- `CAN_CMD_SET_LIMIT_PPOS`
- `CAN_CMD_SET_LIMIT_NPOS`
- `CAN_CMD_SET_MOS_TEMPERATURE`
- `CAN_CMD_SET_OVER_VOLTAGE`
- `CAN_CMD_SET_UNDER_VOLTAGE`

保留这些接口是合理的：

- 后续可以继续只通过参数表使用
- 如需新增专用 UI，可复用 `ProtocolParser::buildSetDriveConfig()`
- 不影响当前控制面板已有功能

## 使用方式

1. 打开工具菜单中的参数表管理器。
2. 导入 `doc/params_20260511_174142.json`。
3. RO 状态项可读取状态。
4. RW 配置项可单项读写或批量写入。
5. 写入 RW 配置项后，主窗口会继续入队 `buildSaveParam()`，用于保存到 EEPROM。

## 已发现问题与修复计划

提交 `486db171` 联调后暴露的 UI 问题及修复策略：

| 问题 | 根因 | 修复 |
| --- | --- | --- |
| 新增 RW 行所有列可编辑 | `updateTableRow()` 默认除 Index 外均可编辑 | 识别 `tx_cmd` 非空的驱动配置行，仅 `Value` 可编辑 |
| Description 显示被截断 | `RxCmd`/`TxCmd` 各占 260px，挤压最后一列 | UI 合并为单列 `Cmd`；`description` 存短名，tooltip 显示完整说明 |
| RxCmd 与 TxCmd 重复 | 50 个 RW 项读写均走同一 `tx_cmd` | UI 显示 `Cmd`；JSON 保留双字段，避免格式迁移 |

执行顺序：先更新本文档，再改 `paramtablewindow.cpp` 与 `params_20260511_174142.json`。

## 风险与注意事项

- `Read All` / `Update` 会触发状态读取和 50 个配置读回，串口流量比纯 RO 状态表更大。
- `Write All` 只会写 `readonly == false` 的参数，状态项不会被误写。
- 保护阈值类参数（过温、过压、欠压、电流/速度/位置限制）需要实机确认合理范围。
- `CAN_CMD_SET_CONTROL_MODE` 同时存在全手设置和单指设置两个入口，测试时需要明确使用的是哪条路径。
- 驱动配置 RW 行的 `Name/Group/Type/Cmd/Offset` 等元数据不应在表格内修改，应通过 JSON 维护。
- 无硬件环境下只能完成构建和 JSON/逻辑检查，实际读写回包、EEPROM 持久化仍需实机验证。

## 验证记录

实施过程中完成过以下验证：

- Python JSON 结构校验通过。
- Qt CMake Debug 构建通过。
- 未做实机联调。

RW 修复后额外检查项：

- 导入参数表后，驱动配置 RW 行仅 `Value` 可双击编辑。
- `Cmd` 列显示正确命令名（如 `CAN_CMD_SET_LIMIT_CURRENT`）。
- `F0_control_mode` 等行的 `Description` 显示短名，悬停可见完整模式枚举。
- 读取/写入仍走 `tx_cmd` + `finger_slot` 路径，不受 UI 列合并影响。

建议实机验证顺序：

1. 单指 `F0_control_mode` 读写。
2. 单指 `F0_limit_cur`、`F0_limit_vel` 读写。
3. 三值参数 `F0_limit_ppos` / `F0_limit_npos` 读写。
4. 五指批量读取。
5. 写入后断电重启确认 EEPROM 持久化。

## 涉及文件

提交 `486db171141cfd2191ac242b7ebd4a8c94e7b1c0` 修改了以下文件：

- `dexterous_hand_ui/doc/params_20260511_174142.json`
- `dexterous_hand_ui/include/mainwindow.h`
- `dexterous_hand_ui/include/paramtablewindow.h`
- `dexterous_hand_ui/include/protocolparser.h`
- `dexterous_hand_ui/src/mainwindow.cpp`
- `dexterous_hand_ui/src/paramtablewindow.cpp`
- `dexterous_hand_ui/src/protocolparser.cpp`
- `dexterous_hand_ui/src/serialcommthread.cpp`
