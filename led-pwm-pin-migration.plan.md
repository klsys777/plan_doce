---
name: led-pwm-pin-migration
overview: 将 GD32F303CB 的三路 LED PWM 固定到 PB3/PB4/PB5，并处理 TIMER1 与系统微秒计数器的资源冲突。
todos:
  - id: timer-resource
    content: 将微秒计数器改为 TIMER3 + 溢出中断软件扩展，释放 TIMER1
    status: completed
  - id: led-pin-map
    content: FACE/PB3 使用 TIMER1_CH1；EAR_LEFT/PB4、EAR_RIGHT/PB5 使用 TIMER2_CH0/CH1
    status: completed
  - id: dma-cleanup
    content: 保持两组 DMA 架构，删除 LED_*_CHCV_ADDR 和重复发送宏
    status: completed
  - id: user-verify
    content: 由用户完成构建、下载和示波器验证
    status: pending
isProject: false
---

# GD32F303CB LED PWM 引脚迁移计划

## 1. 目标与约束

三路 LED 的最终硬件连接：

| 逻辑通道 | 用途 | GPIO | TIMER 通道 | 灯珠 |
|---|---|---|---|---|
| `LED_FACE_ID` | FACE | `PB3` | `TIMER1_CH1` | `HW3528RGB9C-CU3601` |
| `LED_EAR_LEFT_ID` | EAR_LEFT | `PB4` | `TIMER2_CH0` | `HW4516RGB9W-CU5603` |
| `LED_EAR_RIGHT_ID` | EAR_RIGHT | `PB5` | `TIMER2_CH1` | `HW4516RGB9W-CU5603` |

约束：

- 保持现有三路 LED 模块、协议和主循环架构。
- 两侧耳灯继续共用一组 TIMER/DMA，按顺序发送。
- 保留 `USART0` 主通信和 `USART1` 调试串口的 DMA。
- 不修改 Keil 工程用户文件。
- 构建、编译、下载和硬件验证由用户执行。

## 2. GPIO 复用

### FACE / PB3

- `PB3` 对应 `TIMER1_CH1`。
- 使用 `GPIO_TIMER1_PARTIAL_REMAP0`。
- `PB3` 原为 JTAG 引脚；项目的 `bsp_gpio_init()` 已调用
  `GPIO_SWJ_SWDPENABLE_REMAP`，关闭 JTAG 并保留 SWD，因此 PB3/PB4 可作为外设引脚使用。

### EAR_LEFT / PB4 与 EAR_RIGHT / PB5

- `PB4` 对应 `TIMER2_CH0`。
- `PB5` 对应 `TIMER2_CH1`。
- 使用 `GPIO_TIMER2_PARTIAL_REMAP`。

## 3. TIMER1 资源冲突处理

旧实现使用 `TIMER3` 作为微秒计数低 16 位，使用 `TIMER1` 作为级联高 16 位。
由于 FACE/PB3 必须占用 `TIMER1_CH1`，两种配置不能同时存在。

处理方式：

- `TIMER3` 保持 1 MHz、16 位自由运行。
- 启用 `TIMER3` 更新中断，每次溢出增加软件高 16 位。
- `cap_counter_get()` 在短临界区内组合软件高位和 `TIMER3` 计数值。
- 若读取时更新标志已置位但中断尚未执行，读取函数会补偿一次溢出。
- `cap_counter_get()`、`cap_counter_getL16()` 和延时模块的外部接口保持不变。

注意：软件高位依赖溢出中断；如果关闭中断超过一个 `TIMER3` 周期（约 65 ms），可能丢失溢出次数。正常运行和当前短临界区不受影响。

## 4. DMA 映射

| 输出 | DMA 请求 | DMA 通道 | 目标寄存器 |
|---|---|---|---|
| FACE/PB3 | `TIMER1_UP` | `DMA0_CH1` | `TIMER1_CH1CV` |
| EAR_LEFT/PB4 | `TIMER2_UP` | `DMA0_CH2` | `TIMER2_CH0CV` |
| EAR_RIGHT/PB5 | `TIMER2_UP` | `DMA0_CH2` | `TIMER2_CH1CV` |

选择依据：

- `TIMER1_CH1` 的通道事件会使用 `DMA0_CH6`，与 `USART1_TX` 调试串口冲突，因此 FACE 改用 `TIMER1_UP -> DMA0_CH1`。
- `DMA0_CH1` 同时可映射 `USART2_TX`，但当前板级初始化未启用 USART2。
- 两侧耳灯继续通过 `TIMER2_UP -> DMA0_CH2` 轮流发送，不改变现有刷新方式。
- `USART0_RX/DMA0_CH4`、`USART0_TX/DMA0_CH3`、`USART1_RX/DMA0_CH5`、`USART1_TX/DMA0_CH6` 均保持不变。

## 5. LED 时序与颜色顺序

按 `TIMERCLK=120 MHz`、预分频为 0：

| 参数 | FACE / HW3528 | EAR / HW4516 |
|---|---:|---:|
| ARR/period | `108`，约 0.90 us | `144`，约 1.20 us |
| T0H | `36`，约 0.30 us | `36`，约 0.30 us |
| T1H | `72`，约 0.60 us | `108`，约 0.90 us |
| 复位低电平 | 不小于 80 us | 不小于 200 us |
| 颜色顺序 | GRB | RGB，需硬件实测确认 |

## 6. 代码整理

- 删除 `LED_FACE_CHCV_ADDR`、`LED_EAR_LEFT_CHCV_ADDR`、
  `LED_EAR_RIGHT_CHCV_ADDR` 等只使用一次的寄存器地址宏。
- 删除 `LED_FACE_DMA_REQ`，FACE 固定使用 TIMER 更新 DMA 请求。
- 将 `dma_led1_send`、`dma_led2_send`、`dma_led3_send` 三组重复宏合并为
  `dma_led_send(uint8_t led)`。
- DMA 初始化和 TIMER 初始化函数改为按用途命名，避免函数名继续绑定旧引脚和旧通道。
- 删除仅适用于高级定时器 TIMER0/TIMER7 的主输出使能调用；TIMER1/TIMER2 通用定时器不需要该配置。
- 删除 `led.c` 内失效的 `#if 0` 两路 LED 旧实现及无效注释。

## 7. 用户验证清单

### 构建与静态检查

- 构建 APP，并确认无未定义符号或重复中断处理函数。
- 确认启动文件能够链接到 `TIMER3_IRQHandler`。
- 确认 Keil 工程仍使用 GD32F303CB 对应的器件和启动文件。

### 示波器

- PB3：bit 周期约 0.9 us，T0H 约 0.3 us，T1H 约 0.6 us，复位低电平不小于 80 us。
- PB4/PB5：bit 周期约 1.2 us，T0H 约 0.3 us，T1H 约 0.9 us，复位低电平不小于 200 us。
- 验证 PB3、PB4、PB5 均无 JTAG 占用导致的静态电平问题。

### 功能

- 三路灯可独立控制，左右耳不会串路。
- 验证耳灯颜色顺序；若红绿互换，只调整颜色打包顺序，不修改 TIMER/DMA。
- 验证 `periodTimer()`、`delay_us()` 和 `delay_ms()` 长时间运行无明显跳变。
- 连续刷新 FACE 时确认 USART0 通信和 USART1 调试输出正常。
