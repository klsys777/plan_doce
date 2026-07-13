# motor_tool 与 rzt2l_motor_driver 串口协议简表

本文只保留联调常用信息。协议以 `motor_tool` 和 `rzt2l_motor_driver` 当前源码为准，`串口通信协议.xlsx` 仅作参考。

## 1. 基本帧格式

串口参数：`921600, 8N1, no flow control`。

```
PC -> 电机:  AA 55 LEN FRAME_ID MOT_ID PAYLOAD CHK_L CHK_H
电机 -> PC:  5A A5 LEN FRAME_ID MOT_ID PAYLOAD CHK_L CHK_H
```

字段说明：

| 字段 | 长度 | 说明 |
|------|------|------|
| `LEN` | 1 | 数据长度：`FRAME_ID + MOT_ID + PAYLOAD` 的字节数，即 `2 + payload_len` |
| `FRAME_ID` | 1 | 类型/命令 ID |
| `MOT_ID` | 1 | 设备 ID（电机 ID） |
| `PAYLOAD` | 0~126 | 命令数据 |
| `CHK` | 2 | **Checksum**（非 CRC）：对 `LEN + FRAME_ID + MOT_ID + PAYLOAD` 逐字节累加，取 16 位和，小端 |

Checksum 计算（与 `otaThread.cpp::uartPack()` 一致）：

```
chk = 0
for byte in [LEN, FRAME_ID, MOT_ID, PAYLOAD...]:
    chk += byte
CHK_L = chk & 0xFF
CHK_H = (chk >> 8) & 0xFF
```

示例：`AA 55 03 0C 09 C8 E0 00`

| 字节 | 含义 |
|------|------|
| `AA 55` | PC -> 电机帧头 |
| `03` | `LEN=3`，即 `FRAME_ID + MOT_ID + 1 字节 payload` |
| `0C` | `CTRL_SET_MODE` |
| `09` | 电机 ID |
| `C8` | 模式值 200，`MOTOR_RESET` |
| `E0 00` | Checksum，小端；`03+0C+09+C8 = 0x00E0` |

注意：这条复位实际走的是 `0x0C CTRL_SET_MODE + payload 0xC8`，不是单独的 `0x01 MOTOR_RESET`。

## 2. 常用命令

| ID | 名称 | 方向 | 用途 |
|----|------|------|------|
| `0x00` | `ANNOUNCE_DEVID` | 双向 | 扫描设备 |
| `0x01` | `MOTOR_RESET` | 双向 | 复位命令，代码中也常用 `0x0C + 200` |
| `0x02` | `MOTOR_STATE` | 电机 -> PC | 状态帧；UART 侧当前不建议依赖周期上报 |
| `0x03` | `MOTOR_BRAKE` | 双向 | 刹车 |
| `0x04` | `MOTOR_ZERO_MECH` | 双向 | 机械零位 |
| `0x05` | `PARA_STR_INFO` | 电机 -> PC | 参数字符串信息 |
| `0x06` | `PARA_NUM_INFO` | 电机 -> PC | 参数数值信息 |
| `0x07` | `PARA_DONE` | 电机 -> PC | 参数表发送结束 |
| `0x08` | `PARA_UPDATE_FRE` | 双向 | 示波器通道和频率配置 |
| `0x0A` | `PARA_UPDATE_START` | 双向 | 示波器启动/数据上传 |
| `0x0B` | `PARA_UPDATE_STOP` | 双向 | 示波器停止 |
| `0x0C` | `CTRL_SET_MODE` | 双向 | 设置电机运行模式 |
| `0x0D` | `CTRL_SET_PARA` | 双向 | 设置位置/速度/力矩参数，9 个 float |
| `0x0E` | `PARA_READ` | 双向 | 读单个参数 |
| `0x0F` | `PARA_WRITE` | 双向 | 写单个参数 |
| `0x10` | `PARA_SAVE` | 双向 | 保存参数 |
| `0x18` | `CUSTOME_DATA` | 电机 -> PC | 自定义数据，如扫频 |
| `0x19` | `CUSTOME_CMD` | 双向 | 自定义命令 |
| `0x1C~0x1F` | `OTA_*` | 双向 | OTA，经 CAN 桥接；UART 直连固件未完整处理 |

常用模式值：

| 值 | 含义 |
|----|------|
| `200 / 0xC8` | 复位 |
| `3` | 高速端并联闭环 |
| `4` | 高速端串级闭环 |
| `6` | 力矩控制 |
| `7` | 速度控制 |
| `8` | 零位标定 |
| `17` | 电流环扫频 |
| `252` | 齿槽研究 |
| `254` | 电角度标定 |

## 3. 数据类型和参数编码

参数编码：

```
code = (itemType << 12) | posIndex
```

| `itemType` | 含义 |
|------------|------|
| `0` | 可写字符串 |
| `1` | 只读字符串 |
| `2` | 可写参数 |
| `3` | 只读状态参数 |

`paraType` 字节数：

| 值 | 类型 | 字节 |
|----|------|------|
| `0` | U8 | 1 |
| `1` | S8 | 1 |
| `2` | U16 | 2 |
| `3` | S16 | 2 |
| `4` | U32 | 4 |
| `5` | S32 | 4 |
| `6` | FLOAT | 4 |
| `7` | U64 | 8 |
| `8` | S64 | 8 |
| `9` | DOUBLE | 8 |
| `10` | STRING | 16 |

浮点和整型参数按小端内存拷贝传输。参数 `code`、频率等显式字段在现有代码中有大端和小端混用，新增协议应明确写死字节序。

## 4. 常见收发例子

### 4.1 设置复位模式

发送：

```
AA 55 03 0C 09 C8 E0 00
```

含义：对 `MOT_ID=0x09` 发送 `CTRL_SET_MODE(0x0C)`，payload 为 `0xC8`，即 `MOTOR_RESET=200`。

反馈：

```
5A A5 12 0C 09 00 C8 C8 39 47 00 3F 00 00 00 00 33 15 BB BC 11 46 04
```

拆解：

| 字节 | 含义 |
|------|------|
| `5A A5` | 电机 -> PC 帧头 |
| `12` | `LEN=18` |
| `0C` | 回应 `CTRL_SET_MODE` |
| `09` | 电机 ID |
| `00` | `errorCode=0` |
| `C8 C8 ... 11` | 状态和运行数据，按固件 `comPro_sendSetPare()` 的 payload 解析 |
| `46 04` | Checksum，小端；从 `12` 累加到 `11` |

### 4.2 读参数

发送 payload 为 2 字节 `code`：

```
AA 55 04 0E MOT_ID code_hi code_lo CHK_L CHK_H
```

反馈 payload：

```
errorCode code_hi code_lo type data[16]
```

### 4.3 写参数

发送 payload：

```
code_hi code_lo type data[16]
```

反馈 payload：

```
errorCode code_hi code_lo
```

## 5. 现有连续示波器

流程：

```
0x08 配置通道和采样频率
0x0A START
电机持续回 0x0A 数据帧
0x0B STOP
```

`0x08 PARA_UPDATE_FRE` 发送 payload：

| 字段 | 长度 | 说明 |
|------|------|------|
| `chSelNum` | 1 | 通道数 |
| `chSelCode[8]` | 16 | 8 路参数 code，每个 2 字节 |
| `chEchoFreHz` | 2 | 采样频率 Hz |

`0x0A PARA_UPDATE_START` 数据 payload：

```
errorCode timestamp(u16) ch0 ch1 ...
```

各通道数据宽度由 `0x08` 反馈的 `codeType[]` 决定。

## 6. 触发示波器扩展协议

目标：在已有 0x08/0x0A/0x0B 示波器协议上增加触发模式，并支持 A/B 两组条件的单独触发、与关系、或关系。

### 6.1 新增配置命令

建议新增：

```
UART_FRAME_PARA_UPDATE_TRIG_CFG = 0x20
```

发送帧：

```
AA 55 11 20 MOT_ID display_channel pre_trigger trigger_relation trigger_channel_A trigger_condition_A custom_value_A[4] trigger_channel_B trigger_condition_B custom_value_B[4] CHK_L CHK_H
```

payload：

| 偏移 | 字段 | 长度 | 说明 |
|------|------|------|------|
| 0 | `display_channel` | 1 | 显示/采样通道，从 0 开始，决定触发后反馈哪一路数据 |
| 1 | `pre_trigger` | 1 | 预触发比例，单位 %，取值 `0~100` |
| 2 | `trigger_relation` | 1 | 条件关系，从 0 开始 |
| 3 | `trigger_channel_A` | 1 | A 触发对象，按“触发对象”表取值 |
| 4 | `trigger_condition_A` | 1 | A 触发条件，从 0 开始 |
| 5~8 | `custom_value_A` | 4 | A 阈值放大 100 后的 int32，小端 |
| 9 | `trigger_channel_B` | 1 | B 触发对象，按“触发对象”表取值 |
| 10 | `trigger_condition_B` | 1 | B 触发条件，从 0 开始 |
| 11~14 | `custom_value_B` | 4 | B 阈值放大 100 后的 int32，小端 |

条件关系：

| 值 | 关系 | 触发表达式 |
|----|------|------------|
| `0` | `A` | 仅条件 A |
| `1` | `B` | 仅条件 B |
| `2` | `A&&B` | 条件 A 与条件 B 同时成立 |
| `3` | `A||B` | 条件 A 或条件 B 任一成立 |

显示通道：

| 值 | 通道 |
|----|------|
| `0` | `ia` |
| `1` | `ib` |
| `2` | `ic` |
| `3` | `iq` |
| `4` | `id` |
| `5` | `vel` |
| `6` | `pos` |

预触发：

| 字段 | 说明 |
|------|------|
| `pre_trigger` | 预触发占比，单位 `%`。总采样点数固定 `N=1024`：`pre_count = N * pre_trigger / 100`，`post_count = N - pre_count`。上传曲线按时间顺序排列，`seq=0` 为最旧点；触发位置约在 `seq = pre_count`。`pre_trigger=0` 表示无预触发，行为与原先一致（触发后才开始采满 1024 点）；例如 `pre_trigger=25` 时，前 256 点为触发前，后 768 点为触发点及触发后。 |

触发对象：

| 值 | 对象 | 说明 |
|----|------|------|
| `0` | null | 无条件触发。下位机收到任一触发对象为“null”时直接触发，忽略对应 `trigger_condition` 和 `custom_value` |
| `1` | `ia` | A 相电流 |
| `2` | `ib` | B 相电流 |
| `3` | `ic` | C 相电流 |
| `4` | `iq` | q 轴电流 |
| `5` | `id` | d 轴电流 |
| `6` | `vel` | 速度 |
| `7` | `pos` | 位置 |
| `8` | `error` | 错误码触发。上位机输入目标错误码后仍按统一规则执行 `round(input * 100)` 写入 `custom_value`；例如输入 `5` 等待硬件 DRV8353 故障，输入 `1000` 表示任意非 0 错误码都会触发 |

故障/警告错误码候选：

| 值 | 名称 | 说明 |
|----|------|------|
| `0` | - | 无故障/无警告 |
| `1` | `ENC_ERROR_HS` | 高速端编码器无数据 |
| `2` | `ENC_ERROR_LS` | 低速端编码器无数据 |
| `3` | `ERROR_OC` | 软件过流/过载 |
| `4` | `ERROR_OV` | 实际速度超速 |
| `5` | `ERROR_hard` | 硬件 DRV8353 故障 |
| `6` | `ERROR_HVolt` | 软件过压 |
| `7` | `ERROR_LVolt` | 软件欠压 |
| `8` | `ERROR_CALI` | 编码器标定失败 |
| `9` | `ERROR_POSREF` | 位置指令异常超速 |
| `10` | `ERROR_OT` | MTU3 中断持续时间超时 |
| `11` | `ERROR_LACKPHASE` | 缺相故障 |
| `12` | `ERROR_FLYCAR` | 异常飞车 |
| `13` | `ERROR_WrongMotorID` | 电机类型故障 |
| `16` | `ERROR_Motor_OverTemp` | 电机过温故障 |
| `17` | `ERROR_Mos_OverTemp` | MOS 过温故障 |
| `18` | `ERROR_Servo_Invalid` | SERVO 状态切换异常 |
| `19` | `ERROR_Position_Limit` | 位置超限故障 |
| `20` | `ERROR_COgging_W_ERR` | 齿槽转矩写入异常 |
| `21` | `ERROR_kt` | KT 写入结果异常 |
| `256` | `Warning_I2T` | I2T 过热警告 |
| `257` | `ERROR_IMAX` | IMAX 值异常警告 |
| `258` | `ERROR_VMAX` | VMAX 值异常警告 |
| `259` | `ERROR_E2PROM` | E2PROM 读写警告 |
| `260` | `Warning_PHY` | PHY 芯片温度过热警告 |
| `261` | `Warning_Rsampling` | 采样电阻阻值不一致 |
| `262` | `Warning_Block_Wait` | 堵转提示 |


触发条件：

| 值 | 条件 |
|----|------|
| `0` | `>` |
| `1` | `<` |
| `2` | `=` |
| `3` | `>=` |
| `4` | `<=` |

单个条件表达式为：

```
trigger_channel_value trigger_condition custom_value
```

完整触发表达式由 `trigger_relation` 选择 `A`、`B`、`A&&B` 或 `A||B`，显示数据由 `display_channel` 单独输入，预触发窗口由 `pre_trigger` 下发。若触发对象为“null”，下位机收到配置后直接触发，`trigger_condition` 和 `custom_value` 不参与判断。若触发对象为 `error`，上位机输入目标错误码后仍按统一规则执行 `round(input * 100)` 写入 `custom_value`，下位机按缩放前的错误码判断；只有 `motor.warn_faultStatus` 等于该错误码时触发，特殊输入值 `1000` 表示任意非 0 错误码都会触发。例如输入 `5` 时 `custom_value=500`，表示等待硬件 DRV8353 故障；输入 `1000` 时 `custom_value=100000`，表示等待任意故障/警告。

例如显示 `iq`，预触发 `25%`，触发条件为 `iq > 20.00 && vel < 100.00`：`display_channel=3`，`pre_trigger=25`，`trigger_relation=2`，A 条件 `trigger_channel_A=4`、`trigger_condition_A=0`、`custom_value_A=2000`，B 条件 `trigger_channel_B=6`、`trigger_condition_B=1`、`custom_value_B=10000`。上位机输入最多两位小数，下发前执行 `round(input * 100)`，再按 int32 小端写入对应的 `custom_value[4]`。自定义等待不占用 `custom_value`，如需支持应后续单独增加字段或单独配置命令。

### 6.2 触发反馈帧

电机触发后用同一个 `0x20` 分包反馈数据。普通数据帧用 `packet_idx=0~254`，结束帧用 `packet_idx=0xFF`。

```
5A A5 43 20 MOT_ID packet_idx data[64] CHK_L CHK_H
```

普通数据帧 payload：

| 字段 | 长度 | 说明 |
|------|------|------|
| `packet_idx` | 1 | 包序号，从 0 开始递增 |
| `data` | 64 | 底层数组数据，固定 64 字节 |

普通数据帧说明：

- `LEN=0x43`，即 `FRAME_ID(1) + MOT_ID(1) + packet_idx(1) + data[64]`。
- 底层 4096 字节数组按 64 字节分包上传，共 `4096 / 64 = 64` 帧。
- 上位机按 `packet_idx` 拼接 `data[64]`，也可检查丢包或乱序。
- `data[64]` 固定 64 字节，按 **4 字节一组** 解析，每帧共 **16 组**：
- 注意：触发反馈帧已不再携带 `errorCode`，`packet_idx` 后面紧跟 `data[64]`；上位机解析数据时不要预留错误码字节。

| 偏移（组内） | 字段 | 长度 | 说明 |
|--------------|------|------|------|
| 0~1 | `seq` | 2 | 采样序号，`u16`，小端 |
| 2~3 | `value` | 2 | 采样值，`s16`，小端 |

显示换算：

```
display_value = value / 10.0
```

曲线 X 轴建议使用 `seq` 作为横坐标；若某包缺失，可按包内应有 16 组数据补占位点。有预触发时，触发位置约在 `seq = N * pre_trigger / 100`（例如 `pre_trigger=25` 时约在 `seq=256`）。

结束帧：

```
5A A5 03 20 MOT_ID FF CHK_L CHK_H
```

结束帧 payload：

| 字段 | 长度 | 说明 |
|------|------|------|
| `packet_idx` | 1 | 固定 `0xFF`，表示结束帧 |

上位机收到 `packet_idx=0xFF` 后，按已接收的数据帧拼接结果绘制曲线。

### 6.3 触发配置例子

配置 `MOT_ID=1`，预触发 `25%`，触发条件 `iq > 20 && vel < 100`：

| 字段 | 值 |
|------|----|
| `FRAME_ID` | `20` |
| `MOT_ID` | `01` |
| `display_channel` | `03`，即显示 `iq` |
| `pre_trigger` | `19`，即 `25%` |
| `trigger_relation` | `02`，即 `A&&B` |
| `trigger_channel_A` | `04`，即 `iq` |
| `trigger_condition_A` | `00`，即 `>` |
| `custom_value_A` | `D0 07 00 00`，int32 `2000`，即 `20.00 * 100`，小端 |
| `trigger_channel_B` | `06`，即 `vel` |
| `trigger_condition_B` | `01`，即 `<` |
| `custom_value_B` | `10 27 00 00`，int32 `10000`，即 `100.00 * 100`，小端 |

完整帧：

```
AA 55 11 20 01 03 19 02 04 00 D0 07 00 00 06 01 10 27 00 00 69 01
```

触发后反馈第 0 包数据：

```
5A A5 43 20 01 00 data[64] CHK_L CHK_H
```

其中 `00` 是 `packet_idx=0`，后面跟固定 64 字节 `data`。`data` 的解析规则见本节普通数据帧说明。

结束帧示例：

```
5A A5 03 20 01 FF CHK_L CHK_H
```

其中 `FF` 表示结束帧。

## 7. 代码位置

| 功能 | 上位机 | 电机 |
|------|--------|------|
| 封包 | `motor_tool/otaThread.cpp::uartPack()` | `src/UART_RECE_SEND.c::comProtocolDataOut()` |
| 收包 | `motor_tool/otaThread.cpp::getSerialData()` | `src/UART_RECE_SEND.c::uart3_data_receive()` |
| 命令分发 | `motor_tool/otaThread.cpp::analysisRxdDatas()` | `src/UART_RECE_SEND.c` 的 `switch(frame_id)` |
| 示波器 | `motor_tool/otaThread.cpp::scope_dataProcess()` | `src/UART_RECE_SEND.c::echo_main()` |
| 参数表 | `motor_tool/otaThread.h` | `src/parameter.c/h` |

## 8. 当前注意点

1. 上位机接收侧主要按帧头和长度拆包，历史代码里 Checksum 校验不完整；联调时抓包要核对 Checksum。
2. `0x02 MOTOR_STATE` 在固件 UART 侧有发送函数，但当前不应依赖其周期上报。
3. `0x1C~0x1F OTA` 主要走 CAN 桥接，UART 直连固件未完整处理。
4. `RECOVER_FAC(0x11)` 当前固件 UART 侧不是完整恢复出厂实现。
5. 修改 `FRAME_ID`、`paraType`、触发通道编号时，上下位机必须同步。





