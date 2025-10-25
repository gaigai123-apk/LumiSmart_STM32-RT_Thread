## 智能夜灯 + 信息仪表（STM32F103C8T6 · RT-Thread Studio）

本项目基于 RT-Thread（STM32F103C8T6）实现“智能夜灯 + 信息仪表”：

- 自动感知环境明暗，黑暗自动点亮夜灯；明亮熄灭
- 进入/退出黑暗时蜂鸣器提示音（可关闭/手动控制）
- OLED 显示系统信息与当前状态
- 串口 MSH 命令在线配置（自动/取反/蜂鸣器）

### 硬件清单

- STM32F103C8T6 小系统板 ×1（SWD：ST-Link）
- CH340 串口模块 ×1（与 `USART1` 相连）
- 0.96 寸 OLED（SSD1306，I2C） ×1
- 有源蜂鸣器（低电平触发） ×1
- 光敏传感器 ×1（支持数字型或模拟 LDR 分压到 ADC）
- LED ×10（本项目默认使用板载 LED `PC13` 作为夜灯指示）

### 连接与引脚（默认）

- OLED（I2C 软件模拟）
  - `PB8` → SCL
  - `PB9` → SDA
  - VCC → 3.3V，GND → GND
- 蜂鸣器（低电平响）
  - `PB12` → 蜂鸣器负端（或驱动管基极/栅极）
  - 蜂鸣器正端 → 3.3V（电流>20mA 建议加三极管+二极管）
- 光敏传感器（数字型开关量）
  - `PB13`（上拉输入）
- 夜灯 LED（板载）
  - `PC13`（低电平点亮）
- 串口（CH340）
  - `USART1` 115200 8N1（默认 `PA9/PA10`）

如需修改引脚，可在 `applications/main.c` 头部通过宏覆盖：

```c
#define NIGHT_LED_PIN   GET_PIN(C, 13)
// 在对应驱动中也可覆盖：
// #define BUZZER_PIN       GET_PIN(B, 12)
// #define LIGHT_SENSOR_PIN GET_PIN(B, 13)
```

### 主要功能与线程划分

- sensor_thread：采样光敏输入（优先 ADC，若无则数字输入），形成“暗/亮”状态
- control_thread：带时间窗口的迟滞判断；自动控制夜灯与蜂鸣器提示
- ui_thread：OLED 显示系统/状态信息，周期刷新
- MSH 命令：在线查看与配置参数

### 工程位置与关键代码

- 应用入口：`applications/main.c`
- OLED 驱动：`qu__dong/OLED/*`（软件 I2C：PB8/PB9）
- 蜂鸣器：`qu__dong/Buzzer/*`（低电平触发，默认 PB12）
- 光敏传感器：`qu__dong/LightSensor/*`（数字输入，默认 PB13 上拉）

### 构建与下载

1. RT-Thread Studio 打开本工程
2. 确保控制台串口为 `uart1`（`rtconfig.h`：`RT_CONSOLE_DEVICE_NAME "uart1"`）
3. 连接 ST-Link（SWD），CH340（与 `USART1` 连接）
4. Build → Download，打开串口 115200 观察 `msh>` 提示

#### 启用 ADC（可选）

若使用模拟光敏（LDR 分压 → `PA0`）：
- 打开 `drivers/board.h` 中 `#define BSP_USING_ADC1`
- 确保 `drivers/stm32f1xx_hal_conf.h` 开启 `#define HAL_ADC_MODULE_ENABLED`
- `rtconfig.h` 开启 `#define RT_USING_ADC`
- 重新编译下载后，串口执行：
  - `adc probe adc1`
  - `adc enable 0`
  - `night_thr 2000 0`

### 串口 MSH 命令

```text
night_state          # 查看当前状态（原始光敏、电平、自动/取反/蜂鸣器）
night_auto 0/1       # 关闭/开启自动控制（0=手动，1=自动）
night_inv 0/1        # 设置光敏取反（1=常见：暗=0->1；默认 1）
buz 0|1|on|off|tog|pulse
                      # 蜂鸣器：关/开/切换/脉冲提示
night_thr [thr] [ch] # 设置/查看 ADC 阈值与通道（thr:0~4095，ch:0~15）
```

说明：在自动模式下，蜂鸣器仅用于“进入/退出黑暗”的提示音，随后会被自动关闭。

### OLED 显示

- 第一行：`NightLight v1`
- 第二行：`Light:DARK/BRIGHT`
- 第三行：`Auto :ON/OFF`
- 第四行：`Buzz :ON/OFF`（当前蜂鸣器状态）

### 参数与逻辑

- 默认自动模式开启，进入黑暗延时 400ms 判定，进入明亮延时 600ms 判定，避免闪烁抖动
- 自动模式：暗→亮/亮→暗切换时蜂鸣器短促提示；其余时间蜂鸣器保持关闭
- 取反：如果你的光敏模块输出逻辑与预期相反（例如暗时输出低电平），保留取反=1 即可

### 常见问题（FAQ）

1) 我是模拟光敏电阻（LDR）+电阻分压接 ADC，怎么用？

- 工程已内置 ADC 方案：默认打开 `ADC1` 并从 `PA0/ADC1_IN0` 采样，使用命令 `night_thr 2000 0` 设置阈值与通道；若板上无 ADC 驱动或未定义，将自动回退到数字输入方案。

2) OLED 没显示/花屏？

- 检查 `PB8/PB9` 连接与上拉；确认供电 3.3V；若你的 OLED 地址或引脚不同，请在 `qu__dong/OLED/OLED.c` 中调整。

3) 蜂鸣器常响/太响？

- 确保是“低电平触发”的有源蜂鸣器；电流较大时请加驱动管与反灌二极管；也可关闭自动模式：`night_auto 0`，用 `buz` 手动控制。

### 许可证

- RT-Thread 与 STM32 HAL 依据各自上游协议；本项目应用层示例以 Apache-2.0 许可发布。

### 变更记录

- 2025-10-25：初版集成（线程/命令/OLED/UI），数字光敏与蜂鸣器基于 pin 驱动适配。


# LumiSmart_STM32-RT_Thread
