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

目标：在已有 0x08/0x0A/0x0B 示波器协议上增加触发模式。先按简单协议实现，后续再扩展。

### 6.1 新增配置命令

建议新增：

```
UART_FRAME_PARA_UPDATE_TRIG_CFG = 0x20
```

发送帧：

```
AA 55 08 20 MOT_ID trigger_channel trigger_condition custom_value[4] CHK_L CHK_H
```

payload：

| 偏移 | 字段 | 长度 | 说明 |
|------|------|------|------|
| 0 | `trigger_channel` | 1 | 触发通道，从 0 开始 |
| 1 | `trigger_condition` | 1 | 触发条件，从 0 开始 |
| 2~5 | `custom_value` | 4 | 阈值放大 100 后的 int32，小端 |

触发通道：

| 值 | 通道 |
|----|------|
| `0` | `ia` |
| `1` | `ib` |
| `2` | `ic` |
| `3` | `iq` |
| `4` | `id` |
| `5` | `vel` |
| `6` | `pos` |

触发条件：

| 值 | 条件 |
|----|------|
| `0` | `>` |
| `1` | `<` |
| `2` | `=` |
| `3` | `>=` |
| `4` | `<=` |

触发表达式固定为：

```
trigger_channel_value trigger_condition custom_value
```

例如 `iq > 20.00`：`trigger_channel=3`，`trigger_condition=0`，`custom_value=2000`。上位机输入最多两位小数，下发前执行 `round(input * 100)`，再按 int32 小端写入 `custom_value[4]`。自定义等待不占用 `custom_value`，如需支持应后续单独增加字段或单独配置命令。

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

曲线 X 轴建议使用 `seq` 作为横坐标；若某包缺失，可按包内应有 16 组数据补占位点。

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

配置 `MOT_ID=1`，触发条件 `iq > 20`：

| 字段 | 值 |
|------|----|
| `FRAME_ID` | `20` |
| `MOT_ID` | `01` |
| `trigger_channel` | `03`，即 `iq` |
| `trigger_condition` | `00`，即 `>` |
| `custom_value` | `D0 07 00 00`，int32 `2000`，即 `20.00 * 100`，小端 |

完整帧：

```
AA 55 08 20 01 03 00 D0 07 00 00 03 01
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
