# STM32F103 MDK-ARM + VS Code / EIDE HAL 模板

使用 STM32CubeMX 配置 STM32F103C8T6 并生成 MDK-ARM 工程，在 VS Code 中通过 Embedded IDE（EIDE）调用 ARM Compiler 5 编译。本仓库保存了 HAL 驱动、启动文件和 EIDE 工程配置，可作为裸机学习及后续移植 FreeRTOS 的起点。

当前程序完成 HAL、系统时钟和 GPIO 端口时钟初始化后进入空循环，尚未添加 LED 闪烁、串口通信或 FreeRTOS 任务。

## 1. 模板配置

| 项目 | 当前配置 |
| --- | --- |
| MCU | STM32F103C8T6，Cortex-M3，无硬件 FPU |
| Flash | `0x08000000`，`0x10000`（64 KB） |
| RAM | `0x20000000`，`0x5000`（20 KB） |
| 时钟 | 外部 HSE 8 MHz，经 PLL ×9 得到 SYSCLK/HCLK 72 MHz |
| 总线时钟 | APB1 36 MHz，APB2 72 MHz |
| STM32CubeMX | 6.15.0，工程文件为 `Model-MDK.ioc` |
| STM32 固件包 | STM32Cube FW_F1 V1.8.7，使用 HAL |
| EIDE 构建目标 | `Model-MDK` |
| 编译器 | AC5；已用 ARM Compiler 5.06 update 7（build 960）验证 |
| C 语言 | C99 |
| 启动文件 | 根目录的 `startup_stm32f103xb.s`，ARM/Keil 汇编版本 |
| 链接布局 | EIDE 根据 RAM/ROM Layout 自动生成 `.sct` |
| 烧录配置 | 当前选择 J-Link，使用前需要按实际探针补齐配置 |

本模板的 EIDE 工具链是 **AC5，不是 GCC**。MDK-ARM 是 CubeMX 的工程生成选项；实际使用哪个编译器，以 EIDE 的构建配置为准。切换 AC6 或 GCC 时，需要一起核对启动文件、编译选项和链接配置。

## 2. 目录结构

```text
Model-MDK/
├─ Model-MDK.ioc                  CubeMX 硬件配置
├─ Model-MDK.code-workspace       VS Code / EIDE 工作区入口
├─ startup_stm32f103xb.s          当前参与编译的 ARM/Keil 启动文件
├─ .eide/
│  ├─ eide.yml                    源码清单、宏、包含目录、内存及工具链配置
│  └─ files.options.yml           单文件编译选项
├─ .vscode/tasks.json             构建、重新构建和烧录任务
├─ Core/
│  ├─ Inc/                       main.h、gpio.h、HAL 配置头文件等
│  └─ Src/                       main.c、时钟、GPIO、中断及 MSP 初始化
├─ Drivers/
│  ├─ CMSIS/                     内核及 STM32F1 器件支持
│  └─ STM32F1xx_HAL_Driver/       HAL 驱动源码和头文件
├─ MDK-ARM/Model-MDK.uvprojx      CubeMX 生成的 Keil 工程
├─ .cmsis/                       EIDE 附带的 CMSIS 资源
└─ build/Model-MDK/               本机构建产物，已由 Git 忽略
```

EIDE 工程和 VS Code 工作区位于仓库根目录。使用时打开根目录的 `Model-MDK.code-workspace`，不必重新导入 `MDK-ARM` 中的 Keil 工程。

`.eide/eide.yml` 中的 `srcDirs` 为空，是因为此工程通过 `virtualFolder` **逐个列出参与编译的文件**；不代表工程没有源码。`Drivers` 中存在某个文件，也不代表它已自动加入编译。

## 3. 开发环境

准备以下工具：

| 工具 | 用途 |
| --- | --- |
| VS Code | 编辑代码和打开工作区 |
| Embedded IDE，扩展 ID `cl.eide` | 项目管理、构建及烧录入口 |
| Microsoft C/C++，扩展 ID `ms-vscode.cpptools` | 代码补全与导航 |
| ARM Compiler 5 | 提供 `armcc`、`armasm`、`armlink`、`fromelf` |
| STM32CubeMX | 修改 `.ioc` 并重新生成初始化代码 |
| J-Link 软件和探针，或其他已在 EIDE 中配置的烧录工具 | 连接并烧录开发板 |
| Git | 克隆及管理代码 |

在 EIDE 设置中指定本机 ARM Compiler 5 的安装目录，目录下应包含 `bin/armcc.exe` 等工具。仅安装 ARM Compiler 6 或 Arm GCC，不能直接代替当前 AC5 配置。

编译器及烧录工具的安装路径由本机配置，不需要照搬其他电脑上的绝对路径。

## 4. 获取、编译与检查

```powershell
git clone https://github.com/fangzhang299/Model-MDK.git
cd Model-MDK
code Model-MDK.code-workspace
```

没有配置 `code` 命令时，在 VS Code 中选择“文件 → 从文件打开工作区”，选择上述 `.code-workspace` 文件即可。

1. 在 EIDE 面板确认目标为 `Model-MDK`，工具链为 `AC5`，CPU 为 `Cortex-M3`。
2. 确认预处理宏为 `USE_HAL_DRIVER`、`STM32F103xB`。
3. 在“构建配置 → RAM/ROM Layout”中核对以下内容：

   | 区域 | 起始地址 | 大小 | 启用 |
   | --- | --- | --- | --- |
   | IROM1 / Flash | `0x08000000` | `0x10000` | 是，作为启动区域 |
   | IRAM1 / RAM | `0x20000000` | `0x5000` | 是 |
   | 其他区域 | 按实际硬件填写 | 按实际硬件填写 | 本模板不启用 |

4. 执行“重新构建”。正常情况下应编译 **16 个 C 文件和 1 个汇编文件**。
5. 构建产物位于 `build/Model-MDK/`，包括 `.axf`、`.hex`、`.sct`、`.map` 和日志。本模板默认不输出 `.bin`。

发布时已使用 AC5 完成全量构建验证；未据此宣称完成硬件烧录或上板运行测试。参考构建大小为：Code 2132 B、RO-data 304 B、RW-data 16 B、ZI-data 1632 B；不同编译器版本或选项可能使结果变化。

不仅要看“构建成功”，还要检查 `.map` 中的向量表和存储区域：只读代码区域应位于 `0x08000000`，可写数据区域应位于 `0x20000000`。Flash 与 RAM 即使填反，也可能链接成功，因此地址必须核对。

## 5. 启动文件、HAL 配置与中断

### 启动文件

当前参与编译的是仓库根目录的 [startup_stm32f103xb.s](startup_stm32f103xb.s)，它来自 CMSIS 的 `Source/Templates/arm/`，提供中断向量表、`Reset_Handler` 以及进入 C 运行环境的启动代码。

启动文件没有强制的磁盘存放位置，但移动文件后必须同步修改 EIDE 中的引用路径。当前路径相对仓库根目录为 `startup_stm32f103xb.s`。不要使用 GCC 或 IAR 目录中的版本替代 AC5 启动文件，也不要同时编译多个启动文件。

### HAL 配置文件

[Core/Inc/stm32f1xx_hal_conf.h](Core/Inc/stm32f1xx_hal_conf.h) 是当前工程自己的 HAL 配置文件，负责选择 HAL 模块并设置晶振值等参数。头文件目录 `Core/Inc` 已加入 EIDE 的包含目录。

本工程的 HAL 驱动来自仓库内的 STM32CubeF1 文件，不依赖通过 EIDE“芯片支持包”再次安装标准外设库。不要为这个 HAL 工程重复加入 `StdPeriphDrivers`。

### 中断处理文件

[Core/Src/stm32f1xx_it.c](Core/Src/stm32f1xx_it.c) 与 [Core/Inc/stm32f1xx_it.h](Core/Inc/stm32f1xx_it.h) 用于组织异常和中断处理。当前 `SysTick_Handler()` 调用 `HAL_IncTick()`，为 HAL 超时和 `HAL_Delay()` 提供时基，应保留这一调用及该源文件的构建引用。

## 6. 烧录

仓库当前选择 J-Link，但尚未填入可直接用于所有电脑的器件和工具路径。使用前在 EIDE“烧录配置”中选择实际探针并设置本机工具：

- 使用 J-Link 时，选择匹配的 STM32F103C8 器件、SWD 接口及合适的通信速度。
- 使用 ST-Link 时，切换到 EIDE 支持的相应烧录方式，并配置它要求的软件路径。
- 连接 SWDIO/PA13、SWCLK/PA14、GND，并按探针要求连接目标电压参考、板卡电源及可选的 NRST。
- 常规 Flash 烧录起始地址为 `0x08000000`。

完成构建后执行 EIDE 的“烧录”。烧录成功后，当前空循环程序不会产生 LED 闪烁等可见现象；需要自行配置 GPIO 并添加验证代码。

## 7. 用 CubeMX 修改工程

1. 打开 [Model-MDK.ioc](Model-MDK.ioc)，按开发板修改引脚、时钟和外设。当前配置要求外部 HSE 8 MHz；其他晶振或时钟源需要对应调整。
2. 保持需要的 SWD 调试接口配置，`Project Manager` 中的工具链选择与当前工程一致的 MDK-ARM。
3. 将自己的代码放在 CubeMX 的 `USER CODE BEGIN/END` 区域或独立用户文件中，再生成代码。
4. 检查生成后的文件位置，尤其是启动文件。EIDE 当前引用根目录下的启动文件，CubeMX 若更改位置，需要同步引用。
5. 根据新增外设，将对应的 `Core/Src/*.c`、HAL 驱动文件及必要的包含目录补入 EIDE。
6. 执行重新构建并检查日志。

EIDE 导入后的配置与 Keil `.uvprojx` 文件独立，修改 Keil 工程或重新生成 `.uvprojx` 不会自动更新 EIDE 的源码清单。若重新执行导入，要重新核对工作区根目录、相对路径、编译选项及 RAM/ROM 布局。

仓库保留 Keil 工程以便查看或使用，但本 README 的构建验证针对根目录的 EIDE 工程；直接在 Keil 中构建前也应检查其工具链和存储器设置。

## 8. 常见问题

| 现象 | 优先检查 |
| --- | --- |
| `L6236E: No section matches selector` | 启动文件是否存在、是否参与编译，以及是否提供 `RESET` 段 |
| 构建统计显示 0 个汇编文件 | 根目录启动文件是否被删除、移位或排除；是否仍在虚拟文件清单中 |
| 链接成功但程序无法正常启动 | IROM/IRAM 是否填反、向量表地址是否正确、时钟配置是否匹配板卡 |
| 找不到 `stm32f1xx_hal_conf.h` | `Core/Inc` 是否存在于包含目录，工程配置是否被错误迁移 |
| 新外设出现未定义函数 | 对应 HAL 模块是否启用，其 `.c` 文件是否已加入 EIDE |
| 启动或中断符号重复定义 | 是否重复加入启动文件、中断文件或其他库模板 |

EIDE 生成的 `build/Model-MDK/Model-MDK.sct` 属于构建产物。需要修改地址或容量时，修改 RAM/ROM Layout，不要只修改生成的 `.sct`。

## 9. 后续学习 FreeRTOS

本仓库当前是裸机 HAL 模板，没有预置 FreeRTOS。先完成基础程序的编译、烧录和运行，再在独立工程或分支中引入 RTOS。

若保持当前 AC5 + Cortex-M3 工具链，选择匹配版本的 `portable/RVDS/ARM_CM3` 移植层；切换 GCC 时使用对应的 `portable/GCC/ARM_CM3`。还需要正确配置内核源码、`FreeRTOSConfig.h`、内存管理以及 SVC/PendSV/SysTick 处理。使用 CubeMX 生成 RTOS 配置时，应检查 HAL 时基与内核节拍的分配。

## 10. 参考资料与第三方许可

- [EIDE：导入项目](https://em-ide.com/docs/getting-started/import_prj/)
- [EIDE：Keil 项目导入限制](https://em-ide.com/docs/notice/keil_project_limit/)
- [ST 官方 STM32CubeF1](https://github.com/STMicroelectronics/STM32CubeF1)
- [FreeRTOS Cortex-M3 / RVDS 移植层](https://github.com/FreeRTOS/FreeRTOS-Kernel/tree/main/portable/RVDS/ARM_CM3)

第三方代码遵循各自附带的许可，见 [HAL 驱动许可](Drivers/STM32F1xx_HAL_Driver/LICENSE.txt)、[CMSIS 许可](Drivers/CMSIS/LICENSE.txt) 及 [STM32F1 器件支持许可](Drivers/CMSIS/Device/ST/STM32F1xx/LICENSE.txt)。
