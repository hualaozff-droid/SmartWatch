# SmartWatch 固件工程

## 项目简介

SmartWatch 是一套运行在 STM32F411CEU6 上的智能手表固件。项目以低功耗、模块化和可扩展的嵌入式应用为目标，集成了显示、触摸、传感器、按键、电源管理和多种生活工具。

固件由以下几部分组成：

- **硬件抽象与驱动**：基于 STM32 HAL，封装显示屏、触摸、传感器、按键、电源和存储器。
- **图形界面**：使用 LVGL 8.2.0 实现页面、控件、字体、图片和输入设备适配。
- **任务调度**：使用 FreeRTOS，并通过 CMSIS-RTOS2 接口管理应用任务、消息和事件。
- **应用功能**：提供时间、健康数据、环境数据、运动检测、设置和小游戏等页面。

> 本仓库是固件工程，不包含完整的硬件原理图、外壳设计或手机端应用。实际功能取决于连接的硬件版本和对应传感器。

## 功能概览

### 基础功能

- 时间和日期显示
- 日历和时间设置
- 计时器
- 计算器
- 充电状态检测
- 设置保存和读取

### 健康与环境数据

- 心率数据采集
- 血氧相关页面和算法接口
- 温度和湿度显示
- 气压数据显示
- 电子罗盘
- 运动状态和抬腕检测

### 系统与交互

- ST7789 彩色屏幕显示
- CST816 电容触摸
- 板载按键输入
- 屏幕、触摸和传感器的休眠/唤醒
- 通过串口与外部蓝牙模块交互
- 2048、记忆游戏等应用页面

菜单中实际开放的页面由 `User/GUI_App/Screens/Inc/ui_MenuPage.h` 中的宏决定。NFC 卡片页面目前仍包含开发中的占位逻辑，不能视为已经完成的功能。

## 硬件与软件环境

| 项目 | 当前配置 |
| --- | --- |
| 主控 | STM32F411CEU6，Cortex-M4 |
| Flash / RAM | 512 KB Flash / 128 KB RAM |
| 系统时钟 | 100 MHz，详见 `Core/Src/main.c` |
| 显示屏 | ST7789，240 × 280，竖屏 |
| 触摸芯片 | CST816 |
| 温湿度传感器 | AHT21 |
| 气压传感器 | SPL06_001 |
| 电子罗盘 | LSM303DLH |
| 运动传感器 | MPU6050 |
| 心率采集 | EM7028 |
| 非易失存储 | BL24C02 EEPROM |
| 图形库 | LVGL 8.2.0，源码已包含 |
| 实时操作系统 | FreeRTOS，源码已包含 |
| 配置工具 | STM32CubeMX |
| IDE / 编译器 | Keil MDK-ARM 5 / ARM Compiler 5 |

EM7028 目录包含预编译算法库。编译时需要保留 `BSP/EM7028/` 下的 `.lib` 文件，并确保其链接配置与 ARM Compiler 5 兼容。

## 工程结构

```text
smartwatch/
├── BSP/
│   ├── AHT21/            # 温湿度传感器
│   ├── BL24C02/          # EEPROM 与数据保存
│   ├── EM7028/           # 心率采集与算法库
│   ├── LCD/              # ST7789 显示驱动
│   ├── LSM303DLH/        # 电子罗盘
│   ├── MPU6050/          # 运动传感器与 DMP
│   ├── POWER/            # 电源和充电状态
│   ├── SPL06_001/        # 气压传感器
│   ├── TOUCH/            # CST816 触摸驱动
│   └── KEY/、IIC/ 等     # 按键和总线支持
├── Core/
│   ├── Inc/              # CubeMX 生成的头文件
│   └── Src/              # 启动、时钟、外设、中断和 RTOS 接入
├── Drivers/              # CMSIS、STM32F4 HAL 和设备支持文件
├── MDK-ARM/
│   ├── smartwatch.uvprojx # Keil 工程
│   ├── DebugConfig/      # 调试器配置
│   └── output/           # bin、hex 等编译输出
├── Middlewares/
│   ├── Third_Party/      # FreeRTOS
│   └── LVGL/             # LVGL 源码和显示/输入移植
├── SYSTEM/               # 公共系统定义和延时支持
├── User/
│   ├── Func/             # 页面管理、硬件数据和公共功能
│   ├── GUI_App/          # 页面、字体、图片和 LVGL 辅助代码
│   ├── Tasks/            # 应用任务
│   └── version.h         # 固件版本、作者和项目链接
├── smartwatch.ioc        # STM32CubeMX 配置
└── KeilClear.bat         # 清理 Keil 生成文件
```

## 任务与软件架构

系统启动流程大致如下：

1. `Core/Src/main.c` 完成 HAL、时钟和基础外设初始化。
2. `Core/Src/freertos.c` 接入 FreeRTOS 和 CMSIS-RTOS2。
3. `User/Tasks/Src/user_TasksInit.c` 创建应用任务、消息队列和事件标志。
4. `user_HardwareInitTask.c` 初始化显示、输入、传感器和 UI。
5. LVGL 任务持续处理界面刷新和输入事件，其他任务负责按键、传感器、电源和数据保存。

主要任务职责如下：

| 文件 | 职责 |
| --- | --- |
| `user_TasksInit.c` | 创建任务、消息队列和事件标志 |
| `user_HardwareInitTask.c` | 硬件、显示和 UI 初始化 |
| `user_ScrRenewTask.c` | 按键消息、页面返回和传感器休眠 |
| `user_SensUpdateTask.c` | 传感器数据更新和运动状态检测 |
| `user_RunModeTasks.c` | 运行模式和休眠处理 |
| `user_KeyTask.c` | 按键扫描和按键事件 |
| `user_ChargCheckTask.c` | 充电状态检查 |
| `user_DataSaveTask.c` | 设置和运行数据保存 |
| `user_MessageSendTask.c` | 消息和外部通信处理 |

页面生命周期由 `User/Func/Src/PageManager.c` 管理，硬件数据与操作接口集中在 `User/Func/Src/HWDataAccess.c`。新增页面时应优先复用这两个模块，而不是在任务中直接操作页面对象。

## 编译准备

### 必需软件

1. Keil MDK-ARM 5。
2. ARM Compiler 5。工程当前记录的版本为 **V5.06 update 7 (build 960)**，未启用 ARM Compiler 6。
3. `Keil.STM32F4xx_DFP.2.17.1` 器件支持包。
4. STM32CubeMX（仅在需要修改芯片配置时使用）。
5. 与目标板匹配的调试器和下载驱动。

### 编译步骤

1. 使用 Keil 打开 [`MDK-ARM/smartwatch.uvprojx`](MDK-ARM/smartwatch.uvprojx)。
2. 在工程目标中选择 `smartwatch`。
3. 确认 ARM Compiler 5、STM32F411CEUx 器件和头文件路径配置正确。
4. 执行 **Project → Build Target** 或 **Rebuild all target files**。
5. 在 `MDK-ARM/output/` 检查生成的 `smartwatch.bin` 和 `smartwatch.hex`。

工程已包含 FreeRTOS、LVGL、CMSIS 和 STM32 HAL 源码，正常情况下不需要通过包管理器额外下载这些依赖。

## STM32CubeMX 配置

需要修改时钟、GPIO、DMA、SPI、USART 或 FreeRTOS 配置时，可以使用 STM32CubeMX 打开 [`smartwatch.ioc`](smartwatch.ioc)。当前配置记录如下：

- STM32F4 固件包：`STM32Cube FW_F4 V1.27.1`
- 目标工具链：`MDK-ARM V5.32`
- RTOS 接口：CMSIS-RTOS2
- 已配置的主要外设：ADC、DMA、RTC、SPI1、TIM3 和 USART1

重新生成代码前请备份或确认自定义代码区域。生成完成后，必须检查以下内容是否仍然存在：

- 自定义任务创建和消息通信
- LVGL 显示和输入移植
- 中断回调和 DMA 配置
- `SCB->VTOR` 向量表设置
- Keil 工程中的新增源文件、头文件路径和库文件

## 下载地址与启动约束

当前 Keil 应用区域配置为：

```text
Flash 起始地址：0x0800C000
Flash 区域大小：0x00074000
RAM 起始地址：  0x20000000
RAM 区域大小：  0x00020000
```

`Core/Src/main.c` 同时设置了 `SCB->VTOR = 0x0000C000U`。这表示 Flash 前 48 KB 预留给引导程序或其他用途，应用程序不是从默认的 `0x08000000` 开始运行。

烧录时必须保证以下三者一致：

1. Keil 的 IROM 起始地址和应用大小；
2. 链接脚本/目标配置中的应用地址；
3. 引导程序跳转地址以及向量表地址。

如果使用独立 Bootloader，请先确认 Bootloader 会跳转到 `0x0800C000`，并正确设置主栈指针和 `SCB->VTOR`。在 Keil 的 **Options for Target → Debug / Utilities** 中按实际调试器配置连接和下载选项后再烧录。

## 开发指南

### 新增硬件驱动

1. 在 `BSP/` 下建立对应模块目录。
2. 将初始化、读写和休眠接口拆分到 `.c/.h` 文件。
3. 复用 `IIC`、SPI 或 HAL 接口，避免直接修改无关驱动。
4. 在 `smartwatch.uvprojx` 中加入新源文件和头文件路径。
5. 在硬件初始化任务中调用初始化接口。
6. 在传感器任务中通过统一数据接口向 UI 提供数据。

### 新增 UI 页面

1. 在 `User/GUI_App/Screens/` 中添加页面声明、实现和资源。
2. 使用现有 LVGL 页面风格、字体和图片资源。
3. 在页面管理器中注册页面的创建、进入和退出逻辑。
4. 按需修改 `ui_MenuPage.h` 中的菜单宏。
5. 检查页面切换时是否关闭不需要的传感器和定时任务。

### 修改任务

任务之间应使用已有的消息队列、事件标志和数据接口通信。不要在高频传感器任务中执行耗时的 UI 操作，也不要在中断中调用可能阻塞的 FreeRTOS 或 LVGL 接口。

## 调试与常见问题

| 现象 | 排查方向 |
| --- | --- |
| 找不到编译器或器件 | 检查 ARM Compiler 5、STM32F4 器件包和目标配置 |
| EM7028 算法库链接失败 | 检查 `.lib` 文件、库搜索路径和编译器版本 |
| 下载成功但无法启动 | 核对 `0x0800C000` 应用地址、Bootloader 跳转和 `SCB->VTOR` |
| 屏幕全白或花屏 | 检查 ST7789 接线、SPI、DMA、屏幕方向和 LVGL flush 回调 |
| 触摸无响应 | 检查 CST816 的 I2C、复位脚、中断脚和 LVGL 输入回调 |
| 传感器读数异常 | 检查 I2C 地址、供电、初始化顺序和传感器休眠状态 |
| 页面刷新卡顿 | 检查 LVGL tick、刷新任务优先级、DMA 状态和耗时回调 |
| 休眠后无法唤醒 | 检查按键/触摸唤醒源、传感器唤醒接口和电源任务 |
| 设置重启后丢失 | 检查 EEPROM 读写、数据校验和 `user_DataSaveTask` |

调试时建议先确认硬件初始化成功，再逐个启用传感器和页面功能。涉及多个模块的故障，不要只通过修改 UI 页面来规避底层驱动问题。

## 版本信息

固件版本号和版本说明位于 [`User/version.h`](User/version.h)：

```c
#define VERSION_MAJOR 2
#define VERSION_MINOR 4
#define VERSION_PATCH 5
```

版本号规则为：

- `MAJOR`：整体机械结构或重大架构变化
- `MINOR`：硬件功能变化
- `PATCH`：软件修复和小功能更新

## 注意事项

- 本工程当前面向 Keil MDK-ARM 5 和 ARM Compiler 5，直接切换到 ARM Compiler 6 可能需要调整语法、库和链接配置。
- 不要随意将应用地址改回 `0x08000000`，否则可能覆盖 Bootloader 或导致向量表错误。
- 重新生成 CubeMX 代码后必须检查自定义代码和 Keil 工程文件。
- EM7028 的预编译库可能依赖特定编译器和 ABI，升级工具链前应先完成独立验证。
- 实际编译、烧录、传感器精度和功耗表现需要在对应工具链与硬件上验证。

