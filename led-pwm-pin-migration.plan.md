---
name: led-pwm-pin-migration
overview: 迁移三路 LED 输出到 PC9/PC8/暂定脚(当前 PA8)；PC8/PC9 驱动 HW4516，暂定脚驱动 HW3528，按各自数据手册配置时序；暂定脚优先选择不冲突的 DMA 映射，避免动 USART0_RX 的互斥恢复。
todos:
  - id: led-config
    content: 新增灯珠/通道配置宏：路数映射、暂定脚、4516/3528 时序与颜色顺序
    status: pending
  - id: timer-remap
    content: TIMER2 FULL_REMAP 初始化 PC8/PC9；TIMER0_CH0 初始化暂定脚 PA8，各自 period/T0H/T1H
    status: pending
  - id: dma-send
    content: 补齐三路 DMA 发送；暂定脚用不冲突的 TIMER0 触发（当前配置为 CH0→DMA0_CH1），不再依赖 USART0_RX 互斥恢复
    status: pending
  - id: led-module
    content: LED 模块扩为 3 路，按灯珠类型填 buffer（时序值/颜色顺序），加索引保护
    status: pending
  - id: protocol-main
    content: 第三路接入主循环和串口协议，兼容旧 4 字节下发
    status: pending
  - id: verify
    content: make debug/app 编译；整理示波器与协议验证点
    status: pending
isProject: false
---

# LED PWM 引脚迁移计划

> **For agentic workers:** 实施前先核对本计划中的 DMA 映射与灯珠时序；暂定脚相关改动必须集中在配置宏，方便后续换脚。

**Goal:** 把 LED 控制从旧两路 `PB0/PB1` 迁到三路输出，并按实际灯珠型号驱动。

**Architecture:** PC8/PC9 共用 `TIMER2` + `DMA0_CH2`（`TIMER2_UP`）轮流更新 CH2/CH3；暂定脚当前用 `PA8/TIMER0_CH0` 触发 DMA（`TIMER0_CH0 -> DMA0_CH1`），不再触碰 `USART0_RX`（DMA0_CH4）。两款灯珠时序/颜色顺序分配置，不共用一套 `T0H/T1H`。

**Tech Stack:** GD32F303、TIMER PWM + DMA、现有 `led`/`com_protocol` 模块。

## Global Constraints

- 不主动改 `software/keil_project/touch.uvoptx` / `touch.uvprojx` / `touch.uvguix.magiclab`，除非构建必须。
- `USART0` 是主通信口，DMA 通道映射需避开 `DMA0_CH4`（USART0_RX）。暂定脚通过选择不冲突的 TIMER/DMA 映射来避免互斥恢复。
- 暂定脚相关 GPIO/TIMER/DMA 一律走配置宏，后续换脚只改宏，不散改业务代码。
- PC8/PC9 使用 `HW4516RGB9W-CU5603`；暂定脚使用 `HW3528RGB9C-CU3601`。

---

## 1. 目标映射

| 逻辑通道 | 封装脚 / 当前引脚 | TIMER | 灯珠 | 用途 |
|---|---|---|---|---|
| `LED_PWM1` | 暂定 `41/PA8`（后续可换） | `TIMER0_CH0` | `HW3528RGB9C-CU3601` | 头面部 |
| `LED_PWM2` | `40/PC9` | `TIMER2_CH3` | `HW4516RGB9W-CU5603` | 耳朵一侧 |
| `LED_PWM3` | `39/PC8` | `TIMER2_CH2` | `HW4516RGB9W-CU5603` | 耳朵另一侧 |

现状代码：`TIMER2_PARTIAL_REMAP + PB0/PB1`，`dma_ws2812_buffer[2]` 仅两路，`dma_led3_send` 占位。核心文件：

- [bsp_timer.c](../software/GD32F303/BSP/bsp_timer.c)
- [bsp_dma.h](../software/GD32F303/BSP/bsp_dma.h) / [bsp_dma.c](../software/GD32F303/BSP/bsp_dma.c)
- [led.c](../software/Module/led.c) / [led.h](../software/Module/led.h)
- [com_protocol.c](../software/Module/com_protocol.c)
- [BSP_Config.c](../software/GD32F303/BSP/BSP_Config.c)

## 2. 关键结论（来自讨论）

### 2.1 DMA 映射（纠正旧计划）

旧计划写“`TIMER0_CH0` 与 `USART0_TX` 共用 `DMA0_CH3`”不准确。GD32F30x 固定映射：

| 请求源 | DMA 通道 | 本项目占用 |
|---|---|---|
| `TIMER2_UP` | `DMA0_CH2` | 现有 LED，继续给 PC8/PC9 |
| `TIMER0_CH0`（CH0 比较事件） | `DMA0_CH1` | 也映射到 `USART2_TX`（当前应用未启用 USART2） |
| `TIMER0_UP`（更新事件） | `DMA0_CH4` | 已被 `USART0_RX` 占用 |
| `USART0_TX` | `DMA0_CH3` | 主串口发送，勿抢 |

**暂定方案（按你的要求）：** 暂定脚优先选 `TIMER0_CH0` 触发（`TIMER0_CH0 -> DMA0_CH1`），从根源上避开 `TIMER0_UP -> DMA0_CH4`，因此不需要动 `USART0_RX`。

### 2.2 PC8/PC9 共用 `DMA0_CH2` 是否瓶颈

两路共用一个 DMA 通道，**不能同时发**，只能轮流发。每路约 12 灯 × 24bit × ~1.2us ≈ 亚毫秒级；两路顺序刷新通常仍可接受，不是主瓶颈。

### 2.3 灯珠差异

**`HW4516RGB9W-CU5603`（耳朵，PC8/PC9）**

- 供电典型 5V（3.5~5.5V）
- 双输入：`DIN1` 主 / `DIN2` 辅（断点续传）
- 24bit；PDF 文本写 **RGB** 顺序（高位先发）→ **必须实测确认**，与当前代码 GRB 可能不一致
- 时序典型：`T0H=0.3us`，`T1H=0.9us`，`T0L=0.9us`，`T1L=0.3us`，`Trst>=200us`
- 现有 `T1H=72(~0.6us)` **偏短**，不能直接复用 3528/旧参数

**`HW3528RGB9C-CU3601`（头面部，暂定脚）**

- 4pin：`VDD/DO/DIN/GND`
- 24bit，明确 **GRB**（与当前 `led.c` 一致）
- 时序按手册：`T0H=0.2~0.35us(typ 0.3)`，`T1H=0.5~0.85us(typ 0.6)`，`T0L=0.5~0.85us(typ 0.6)`，`T1L=0.2~0.35us(typ 0.3)`，`Trst>=80us`
- 建议 bit 周期约 **0.9us**（`period≈108` @ 120MHz PSC=0），不要继续用旧 `period=150`（否则 `T0L/T1L` 会超 3528 手册上限）
- 注意：`VIH>=0.7*VDD`；若灯珠 5V 供电而 MCU 3.3V，可能需要电平转换或实测容忍度

### 2.4 推荐 TIMER 时序参数（@ TIMERCLK=120MHz，PSC=0）

| 参数 | HW3528（暂定脚 / TIMER0） | HW4516（PC8/PC9 / TIMER2） |
|---|---|---|
| `period`（ARR） | `108`（~0.90us） | `144`（~1.20us） |
| `T0H` | `36`（~0.30us） | `36`（~0.30us） |
| `T1H` | `72`（~0.60us） | `108`（~0.90us） |
| 复位低电平 | `>=80us` | `>=200us` |
| 颜色顺序 | `GRB` | 暂按手册 `RGB`，加宏；实测后可改 |

发送结束后比较寄存器置 0，并保证复位低电平时间满足对应灯珠。

## 3. 暂定脚配置策略

在 `led.h` 或新建轻量配置头（优先放 `led.h` / `bsp_timer.h`，避免扩散）集中定义，例如：

```c
/* 暂定：头面部 LED 输出脚，后续换脚只改这里 */
#define LED_FACE_GPIO_PORT        GPIOA
#define LED_FACE_GPIO_PIN         GPIO_PIN_8
#define LED_FACE_TIMER            TIMER0
#define LED_FACE_TIMER_CH         TIMER_CH_0
#define LED_FACE_DMA              DMA0
#define LED_FACE_DMA_CH           DMA_CH1
#define LED_FACE_DMA_REQ          TIMER_DMA_CH0D   /* TIMER0_CH0 -> DMA0_CH1 */
#define LED_FACE_CHCV_ADDR        ((uint32_t)&TIMER_CH0CV(TIMER0))

#define LED_FACE_PERIOD           108
#define LED_FACE_T0H              36
#define LED_FACE_T1H              72
#define LED_FACE_COLOR_ORDER_GRB  1
#define LED_FACE_RESET_US         80

#define LED_EAR_PERIOD            144
#define LED_EAR_T0H               36
#define LED_EAR_T1H               108
#define LED_EAR_COLOR_ORDER_RGB   1
#define LED_EAR_RESET_US          200
```

约定：

- `LED_PWM1` = face = 暂定脚宏
- `LED_PWM2` = ear on PC9 / TIMER2_CH3
- `LED_PWM3` = ear on PC8 / TIMER2_CH2
- 后续换脚：改 GPIO/TIMER/DMA 宏，并重新核对新脚的 DMA 请求映射；**不要**再默认假设占 `DMA0_CH4`

## 4. 实现步骤

### Task A — 配置与灯珠参数

- [ ] 在 [led.h](../software/Module/led.h)（或共用头）增加上表宏：`LED_NUM` 暂维持 12，除非硬件确认颗数不同。
- [ ] 去掉全局单一 `T0H/T1H`，改为按通道取 `LED_FACE_*` / `LED_EAR_*`。
- [ ] 颜色填充函数按通道选择 GRB/RGB。

### Task B — TIMER 初始化

- [ ] 改 [bsp_timer.c](../software/GD32F303/BSP/bsp_timer.c)：
  - `TIMER2`：`GPIO_TIMER2_FULL_REMAP`，`GPIOC PIN_8|PIN_9`，CH2/CH3 PWM，`period=LED_EAR_PERIOD`，`timer_dma_enable(TIMER2, TIMER_DMA_UPD)`。
  - 新增 `TIMER0` 初始化：按 `LED_FACE_*` 配 PA8/CH0，`period=LED_FACE_PERIOD`，`timer_dma_enable(TIMER0, TIMER_DMA_CH0D)`。
- [ ] 在 [bsp_timer.h](../software/GD32F303/BSP/bsp_timer.h)、[BSP_Config.c](../software/GD32F303/BSP/BSP_Config.c) 声明并调用。

### Task C — DMA 发送

- [ ] 保留 `DMA0_CH2` 给 TIMER2 两路耳朵灯：发送时切 `PADDR` 到 `TIMER2_CH3CV` / `TIMER2_CH2CV`。
- [ ] 新增 face 发送路径：使用 `LED_FACE_DMA_CH`（当前 `DMA0_CH1`），`PADDR=LED_FACE_CHCV_ADDR`，`MADDR=face buffer`。
- [ ] face 发送时不再需要互斥恢复 USART0_RX：仅使用 `DMA0_CH1` 将 buffer 写入 `TIMER0_CH0CV`，并在发送结束后按 `LED_FACE_RESET_US` 满足复位低电平时间。
- [ ] 明确注释：此为暂定；换脚后优先选不撞 `USART0_RX` 的 TIMER/DMA 组合。

### Task D — LED 模块 3 路

- [ ] [led.c](../software/Module/led.c)：`dma_ws2812_buffer[3][...]`，`oldCMD/oldClr` 扩到 3，`ws2812SetColor`/`setLEDAction` 索引 0/1/2。
- [ ] 填 buffer 时写入对应灯珠的 `T0H/T1H`；颜色按通道顺序。
- [ ] 发送后 delay 使用对应 `RESET_US`（4516 至少 200us，3528 至少 80us）。
- [ ] 越界保护：非法索引直接 return。

### Task E — 协议与主循环

- [ ] [main.c](../software/User/main.c)：第三路 `setLEDAction(2, ...)`。
- [ ] [com_protocol.c](../software/Module/com_protocol.c)：收发增加 `clr3/cmd3`；旧 4 字节下发 `len<6` 时 LED3 保持默认关或上次状态。

## 5. 验证计划

- 编译：`software/` 下 `make debug`，必要时 `make app`。
- 静态：无两路写死残留（`buffer[2]`、`oldCMD[2]` 等）。
- 示波器：
  - PC9/PC8：确认 `T1H≈0.9us`、bit≈1.2us、复位≥200us；验证 RGB 颜色是否正确（错则改顺序宏）。
  - 暂定脚 PA8：确认 `T0H≈0.3us`、`T1H≈0.6us`、bit≈0.9us、复位≥80us；GRB 颜色正确。
- 功能：串口协议独立控制三路；发 face 灯时不占用 `USART0_RX`（DMA0_CH4），USART0 接收正常。
- 后续换脚检查清单：新脚 AF 功能、TIMER 通道、DMA 请求映射、是否仍撞 USART0、宏是否一处改完。

## 6. 明确不做 / 暂缓

- 不把 USART0_TX 改成非 DMA（旧备选）。
- 不把 face 改成软件 bit-bang（除非换脚后仍无可用 DMA）。
- 不在本阶段实现 4516 的 DIN2 断点续传硬件冗余逻辑（单线 DIN1 足够先通）。
- 不主动改 Keil 工程用户文件。
