# STM32G474 四开关同步整流 Buck-Boost 数字电源 (v2.0)

基于 **STM32G474RET6** 的四开关同步整流 Buck-Boost 数字可调电源，包含完整的功率主回路、MOSFET 驱动电路、信号调理采样电路、多级辅助电源以及数字闭环控制固件，支持 DC 接口输入与恒压（CV）/ 恒流（CC）自动切换。

---

## 设计指标

| 参数 | 指标 |
| :--- | :--- |
| **主控 MCU** | STM32G474RET6（Cortex-M4F @ 170MHz，内置高分辨率定时器 HRTIM） |
| **输入电压范围** | 12V ~ 30V DC（支持 DC 座及接线端子输入） |
| **输出电压范围** | 0.5V ~ 30V DC 连续可调 |
| **最大输出电流** | 5A |
| **最大输出功率** | 100W |
| **开关频率** | 181.333 kHz（HRTIM 32 倍频，周期 30000，硬件死区约 73.5ns） |
| **目标转换效率** | > 94% |
| **输出电压纹波** | < 200 mV |

---

## 项目结构

```text
buck-boost2.0/
├── Buck-Boost.ioc              # STM32CubeMX 工程配置文件
├── README.md                   # 项目说明文档
├── Core/
│   ├── Hardware/               # 板载外设驱动
│   │   ├── Key.c / Key.h       # 独立按键与编码器按键状态机消抖驱动
│   │   ├── OLED.c / OLED.h     # 0.96寸 OLED 显示驱动（硬件 I2C3）
│   │   ├── OLED_Data.c / .h    # OLED 中英文字模数据
│   │   └── W25Q64.c / .h       # W25Q64 SPI Flash 驱动（掉电参数存储）
│   ├── Inc/                    # 应用与外设头文件
│   │   ├── PID.h               # 电压/电流双环 PID 接口与周期定义
│   │   ├── function.h          # 状态机、采样、保护、界面等核心声明与宏定义
│   │   └── ...                 # HAL 外设配置头文件
│   └── Src/                    # 应用与外设源文件
│       ├── main.c              # 系统时钟、外设初始化、主循环与定时中断调度
│       ├── PID.c               # CCMRAM 加速的电压/电流双环 PID 闭环控制
│       ├── function.c          # 状态机、软启动、模式切换、采样滤波、保护与 UI
│       ├── hrtim.c             # HRTIM1 Timer D / F 互补 PWM 与死区配置
│       ├── adc.c               # ADC1(DMA+4x过采样) / ADC2(NTC) / ADC5(MCU温度)
│       ├── usart.c             # USART1 DMA 串口打印（921600bps）
│       └── ...                 # TIM / SPI / I2C / IWDG / GPIO 初始化
├── Drivers/                    # STM32G4xx HAL 驱动库与 CMSIS
└── MDK-ARM/                    # Keil MDK-ARM v5 工程目录
    ├── Buck-Boost.uvprojx      # Keil 工程文件
    └── startup_stm32g474xx.s   # 启动文件
```

---

## 硬件架构

### 1. 主功率电路（四开关同步整流 Buck-Boost）

```text
           Q1 (Buck 上管)          Q3 (Boost 上管)
VIN+ ──┬──────┤├───┬──── L (22uH) ────┬───┤├──────┬── VOUT+
       │           │                  │           │
       │      Q2 (Buck 下管)     Q4 (Boost 下管)  │
       │      ┤├───┴─── PGND ─────────┴───┤├      │
      R11 (5mΩ)                        R12 (5mΩ)
```

- **功率 MOSFET**：`CJAC80SN10` × 4（100V / 80A）
- **半桥驱动芯片**：`EG3112` × 2（`U8` 驱动 Buck 桥臂 `PB14/PB15`，`U9` 驱动 Boost 桥臂 `PC6/PC7`）
- **储能电感**：`22µH` 功率电感
- **输入保护**：自恢复保险丝 `2920L500/30GR` + 反接保护二极管 `SS56`

### 2. 信号采样与调理电路

| 采样信号 | 硬件方案 | MCU 通道 | 采样方式 |
| :--- | :--- | :--- | :--- |
| **输入电压 `VIN`** | 100kΩ / 10kΩ 分压 + RC 滤波 | `ADC1_IN1 (PA0)` | HRTIM 触发 + DMA + 4× 硬件过采样 |
| **输入电流 `IIN`** | 5mΩ 检流电阻 + `GS8558-SR` 100 倍差分放大 | `ADC1_IN2 (PA1)` | HRTIM 触发 + DMA + 4× 硬件过采样 |
| **输出电压 `VOUT`** | 100kΩ / 10kΩ 分压 + RC 滤波 | `ADC1_IN3 (PA2)` | HRTIM 触发 + DMA + 4× 硬件过采样 |
| **输出电流 `IOUT`** | 5mΩ 检流电阻 + `GS8558-SR` 100 倍差分放大 | `ADC1_IN4 (PA3)` | HRTIM 触发 + DMA + 4× 硬件过采样 |
| **主板温度** | 10kΩ NTC（B=3950）+ 10kΩ 上下拉分压 | `ADC2_IN17 (PA4)` | 软件触发 + 256× 硬件过采样 |
| **MCU 温度** | STM32G4 内部温度传感器 | `ADC5_TEMP` | 软件触发 + 64× 硬件过采样 + 低通滤波 |

### 3. 辅助电源电路

```text
VIN (12~30V) ──▶ TPS54360B (12V，供驱动与风扇)
                    └──▶ TPS5430 (5V，供 OLED 与 CH340C)
                            ├──▶ AMS1117-3.3 (3.3V，供 MCU 与外设)
                            └──▶ REF3033 (3.3V 高精度基准，供 VREF+、运放与 NTC)
```

---

## 固件核心功能

### 1. 闭环控制与工作模式
- **高速中断环路**：`HRTIM1_TIMD_IRQHandler` 周期执行 `BuckBoostVILoopCtlPID()`（放置在 **CCMRAM** 中运行以降低延迟）。
- **双环 CV/CC 控制**：电流环动态修正电压参考值，实现恒压（CV）与恒流（CC）自动平滑切换；电压环与电流环均具备抗积分饱和限幅。
- **三模式自动切换（带滞回）**：
  - **Buck 模式**：`Vout_ref < 0.8 × Vin`（Boost 上管 94%、下管 6% 维持自举充电）
  - **Mix 模式**：`0.85 × Vin ~ 1.2 × Vin` 附近（Buck 固定 80%，Boost 闭环调节）
  - **Boost 模式**：`Vout_ref > 1.2 × Vin`（Buck 固定 94%，Boost 闭环调节）
- **软启动机制**：`StateMRise()` 分 `SSInit → SSWait → SSRun` 三阶段，从最小占空比渐进放开最大占空比限幅，抑制启动浪涌。

### 2. 保护功能

| 保护类型 | 触发条件 | 动作与恢复 |
| :--- | :--- | :--- |
| **输出过压保护 (OVP)** | `Vout ≥ MAX_VOUT_OVP_VAL`（默认 `31.0V`，可调）持续 10ms | 立即关断 HRTIM PWM 输出，蜂鸣器报警并显示故障 |
| **输出过流保护 (OCP)** | `Iout ≥ MAX_VOUT_OCP_VAL`（默认 `5.5A`，可调）持续 50ms | 立即关断 PWM，等待 2s 自动尝试恢复（最多 10 次）；正常稳态运行 5s 或按键清错后复位重试计数 |
| **输出短路保护** | `Iout > 6.0A` 且 `Vout < 0.5V` | 立即关断 PWM，支持有限次数自动恢复及锁死保护 |
| **过温保护 (OTP)** | 主板 NTC 温度 `≥ MAX_OTP_VAL`（默认 `80℃`，可调） | 立即关断 PWM 并报警 |
| **独立看门狗 (IWDG)** | 主循环定时喂狗 | 防止程序跑飞死锁 |

### 3. 人机交互与辅助功能
- **OLED 多页面显示**（0.96 寸 I2C，100ms 刷新）：
  1. **设置页**：电压/电流设定值、实时输出电压/电流、CV/CC 模式圆点指示
  2. **监测页 1**：实时输入电压/电流、输出电压/电流
  3. **监测页 2**：MCU 温度、主板 NTC 温度、实时转换效率
  4. **保护设置页**：过温 / 过流 / 过压保护阈值调节
  5. **故障页**：发生保护时自动切换并显示对应故障类型
- **按键与编码器操作**：
  - **旋转编码器**：未选中设置项时切换显示页面；选中某一位后增减数值
  - **KEY1 (`PA6`)**：在设置页/保护设置页切换选中项；故障状态下清除故障
  - **KEY2 (`PA7`)**：开启 / 关闭电源输出；故障状态下清除故障
  - **编码器按键 (`PC2`)**：切换调节位（十位 / 个位 / 小数第一位 / 小数第二位）
- **参数掉电保存**：使用 `W25Q64` SPI Flash 自动保存设定电压/电流及保护阈值，开机读取并做合法性校验。
- **温控风扇**：根据主板 NTC 温度在 `35℃ ~ 65℃` 区间通过 `TIM8_CH3` PWM 自动分级调速（0% ~ 100%）。
- **串口上位机输出**：`USART1`（`921600bps`，DMA 非阻塞发送），每 50ms 输出一帧 CSV 数据（兼容 VOFA+）：
  ```text
  VIN,IIN,VOUT,IOUT,MainBoard_TEMP,CPU_TEMP,powerEfficiency,CVCC_Mode,raw:ADC0,ADC1,ADC2,ADC3
  ```

---

## 开发与编译环境

- **IDE / 编译器**：Keil MDK-ARM v5（ARM Compiler 6 / ARMCLANG）
- **初始化工具**：STM32CubeMX（`Buck-Boost.ioc`）
- **烧录调试**：SWD 接口（ST-Link / J-Link / DAPLink）

### 编译与烧录步骤
1. 使用 Keil MDK 打开 `MDK-ARM/Buck-Boost.uvprojx`。
2. 点击 **Build** 编译生成 `MDK-ARM/Buck-Boost/Buck-Boost.axf` 与 `.hex`。
3. 连接 SWD 调试器，点击 **Download** 烧录至 STM32G474RET6。

---

## 开源参考

本项目固件框架参考自 [zeruns/STM32-Buck-Boost](https://gitee.com/zeruns/STM32-Buck-Boost) 开源项目，并结合 2.0 硬件方案进行了电路适配、保护修复与控制优化。
