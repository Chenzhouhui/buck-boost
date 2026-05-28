# STM32G474 四开关 Buck-Boost 数字电源

基于 STM32G474RET6 的同步整流四开关 Buck-Boost 数字电源，支持 PD 快充输入和 DC 接口输入。

## 项目概述

- **MCU**: STM32G474RET6（170MHz Cortex-M4F，内置 HRTIM）
- **输入电压**: 12V-48V DC（DC 接口 / Type-C PD 诱骗）
- **输出电压**: 0.5V-48V 可调
- **输出电流**: 0-10A
- **最大功率**: 450W
- **开关频率**: 181.333kHz
- **实测效率**: 最高 94.3%

## 项目结构

```
buck-boost/
├── Buck-Boost.ioc          # STM32CubeMX 工程文件
├── LICENSE
├── Core/
│   ├── Hardware/           # 硬件驱动层
│   │   ├── Key.c/h         # 按键驱动（旋转编码器 + 按钮）
│   │   ├── OLED.c/h        # OLED 显示驱动（I2C）
│   │   ├── OLED_Data.c/h   # OLED 字模数据
│   │   └── W25Q64.c/h      # SPI Flash 驱动（参数存储）
│   ├── Inc/                # 头文件
│   │   ├── PID.h           # PID 算法接口
│   │   ├── function.h      # 功能函数接口
│   │   ├── hrtim.h         # HRTIM 配置
│   │   └── ...             # HAL 外设头文件
│   └── Src/                # 源文件
│       ├── main.c          # 主程序入口
│       ├── PID.c           # PID 闭环控制算法
│       ├── function.c      # 核心功能（状态机、采样、保护）
│       ├── hrtim.c         # HRTIM PWM 配置
│       ├── adc.c           # ADC DMA 采样配置
│       └── ...             # HAL 外设初始化
├── Drivers/                # STM32 HAL 库和 CMSIS
│   ├── CMSIS/              # ARM CMSIS 核心头文件
│   └── STM32G4xx_HAL_Driver/  # STM32G4 HAL 驱动库
└── MDK-ARM/                # Keil MDK 工程
    ├── Buck-Boost.uvprojx  # Keil 工程文件
    ├── Buck-Boost.uvoptx   # Keil 工程选项
    └── startup_stm32g474xx.s  # 启动文件
```

## 核心模块说明

### 1. 主功率电路（四开关 Buck-Boost）

```
        Q1(高边)          Q3(高边)
Vin ──┬──┤├──┬── L ──┬──┤├──┬── Vout
      │       │       │       │
      └──┤├──┘       └──┤├──┘
        Q2(低边)          Q4(低边)
```

- **Buck 模式**: Q1+Q4 工作，Q2/Q3 常关/常开
- **Boost 模式**: Q3+Q2 工作，Q1/Q4 常开/常关
- **Buck-Boost 模式**: 四管交替工作（Vin ≈ Vout 时）
- **MOSFET**: CJAC80SN10（100V/80A，Rds(on)=6.2mΩ）
- **驱动**: EG3112 半桥驱动（内置死区 ~100ns）

### 2. 数字控制（PID 闭环）

```c
// 每个 PWM 周期执行一次（181.333kHz）
① ADC 采样 → Vin, Vout, Iin, Iout, Temp
② 模式判断 → Buck / Boost / Buck-Boost
③ 保护判断 → 过压/过流/过温/短路
④ PID 计算 → error = Vset - Vout
              duty = Kp*error + Ki*∫error + Kd*d(error)/dt
⑤ PWM 更新 → 设置 HRTIM 占空比
```

- **PID 参数**: 支持恒压模式和恒流模式
- **积分限幅**: 防止积分饱和
- **输出限幅**: 占空比限制在安全范围

### 3. 采样电路

| 采样对象 | 方案 | 精度 |
|---------|------|------|
| 输入/输出电压 | 差分放大（GS8558-SR） | 4.7kΩ/75kΩ 分压 |
| 输入/输出电流 | 5mΩ 采样电阻 + 差分放大 | 增益 62 倍 |
| 主板温度 | NTC 10kΩ/B3950 + 分压 | ±1°C |
| MCU 温度 | 内置温度传感器 | ±2°C |

### 4. 辅助电源

```
输入电压 → TPS54360B(12V) → SY8205(6V) → AMS1117-5(5V)
                                          → AMS1117-3.3(3.3V)
                                          → REF3033(3.3V 参考)
```

### 5. 人机交互

- **OLED 显示**: 0.96 寸 I2C 接口，显示电压/电流/功率/温度
- **旋转编码器**: 调节输出电压和电流设置
- **按键**: 切换页面、开启/关闭输出
- **蜂鸣器**: 操作反馈和报警
- **串口**: USART1 通过 CH340C 转 USB，支持 VOFA+ 上位机

### 6. 保护功能

| 保护类型 | 检测方式 | 阈值 | 响应 |
|---------|---------|------|------|
| 输出过压 | ADC 采样 | 可设置 | 立即关断 PWM |
| 输出过流 | ADC 采样 | 可设置 | 立即关断 PWM |
| 过温保护 | NTC 采样 | 可设置（默认 80°C） | 降功率/关断 |
| 短路保护 | 电流>10.1A 且电压<0.5V | 固定 | 立即关断 |
| 输入欠压 | ADC 采样 | <10V | 关断输出 |

### 7. 参数存储

使用 W25Q64 SPI Flash 存储：
- 输出电压/电流设置值
- 保护阈值设置
- 校准参数

掉电不丢失，开机自动恢复。

## 开发环境

- **IDE**: Keil MDK-ARM v5
- **CubeMX**: 用于外设初始化配置
- **HAL 库**: STM32G4xx HAL Driver
- **调试**: SWD（J-Link / ST-Link）

## 编译与烧录

1. 用 Keil MDK 打开 `MDK-ARM/Buck-Boost.uvprojx`
2. 点击 Build 编译
3. 连接调试器，点击 Download 烧录

## 关键数据

| 测试条件 | 输入 | 输出 | 效率 |
|---------|------|------|------|
| 20V→15V/5A | 80.7W | 75.1W | 93.0% |
| 48V→36V/10A | 378W | 357W | 94.3% |
| 36V→24V/9A | 231W | 216W | 93.5% |
| 20V→5V/9A | 52.9W | 44.7W | 84.5% |

输出纹波: 36V输入、12V/2A输出时，峰峰值约 42mV。

## 原始项目来源

本项目基于 [zeruns](https://gitee.com/zeruns/STM32-Buck-Boost) 的开源项目，原始仓库包含完整的原理图、PCB、外壳 3D 模型等资料。
