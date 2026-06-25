# 用 VS Code 从零搭建 STM32 开发环境
这个过程会让你清楚地看到 Keil 替你做了什么。

---

## 第一步：安装工具链
```bash
# Keil 内置了 ARM 编译器 (ARMCC/ARMCLANG)
# 离开 Keil，你需要手动安装：
arm-none-eabi-gcc   # 编译器
arm-none-eabi-ld    # 链接器
arm-none-eabi-objcopy  # 格式转换（elf → hex/bin）
arm-none-eabi-size     # 查看内存占用
```

**Keil 帮你做的：** 安装时自带编译器，你从不需要关心编译器在哪、版本是什么。

---

## 第二步：准备启动文件和链接脚本
```bash
你的项目/
├── startup_stm32f103xx.s   ← 汇编启动文件（Keil模板自动添加）
├── STM32F103XB_FLASH.ld    ← 链接脚本（Keil的.sct等价物）
├── stm32f1xx.h             ← 寄存器头文件
└── main.c
```

+ **启动文件**：初始化堆栈、向量表，跳转到 `main()`
+ **链接脚本**：告诉链接器 Flash/RAM 地址和大小

**Keil 帮你做的：** 新建项目时自动选芯片、自动添加 `startup_xxx.s`，`.sct` 文件自动生成，你几乎看不见它们。

---

## 第三步：手写 Makefile（或 CMakeLists.txt）
这是**最能体现 Keil 做了什么**的一步：

```bash
# Makefile 核心内容

# 1. 定义工具链
CC      = arm-none-eabi-gcc
OBJCOPY = arm-none-eabi-objcopy

# 2. 编译选项（Keil的"Target"选项卡里那些勾选项）
CFLAGS  = -mcpu=cortex-m3 -mthumb       # 指定CPU架构
CFLAGS += -O1                            # 优化等级
CFLAGS += -Wall
CFLAGS += -DSTM32F103xB                  # 宏定义（Keil C/C++选项卡里填的）
CFLAGS += -I./include                    # 头文件路径（Keil里右键Add Path）

# 3. 链接选项
LDFLAGS = -T STM32F103XB_FLASH.ld       # 指定链接脚本
LDFLAGS += -nostdlib                     # 不用标准库（嵌入式常见）

# 4. 编译 → 目标文件
main.o: main.c
    $(CC) $(CFLAGS) -c main.c -o main.o

# 5. 链接 → elf
firmware.elf: main.o startup.o
    $(CC) $(LDFLAGS) main.o startup.o -o firmware.elf

# 6. 转换格式 → hex（Keil Build完自动生成的）
firmware.hex: firmware.elf
    $(OBJCOPY) -O ihex firmware.elf firmware.hex
```

**Keil 帮你做的：** 上面这几百行配置，Keil 通过图形界面全帮你管理了，点击 "Build" 就自动完成所有步骤。

---

## 第四步：配置烧录
```bash
# Keil 内置了 J-Link/ST-Link 烧录支持，点Download就完事
# 离开 Keil，你需要：

# 方式一：OpenOCD（开源）
openocd -f interface/stlink.cfg \
        -f target/stm32f1x.cfg \
        -c "program firmware.elf verify reset exit"

# 方式二：STM32CubeProgrammer CLI
STM32_Programmer_CLI -c port=SWD -d firmware.hex -rst
```

---

## 第五步：配置调试（GDB）
```bash
// VS Code 的 launch.json
{
  "type": "cortex-debug",
  "request": "launch",
  "servertype": "openocd",      // Keil 内置调试服务器
  "executable": "firmware.elf",
  "configFiles": ["interface/stlink.cfg", "target/stm32f1x.cfg"]
}
```

---

## 总结：Keil 替你做了什么
```bash
你在 Keil 的操作                    实际发生的事
─────────────────────────────────────────────────────
选择芯片型号             →   自动添加 startup.s + 生成 .sct 链接脚本
在Target选项卡填Flash/RAM →   配置链接脚本中的地址和大小
添加源文件到项目          →   告诉 Makefile 哪些 .c 需要编译
Include Paths 填路径      →   -I 参数
Define 填宏              →   -D 参数
选 Optimization Level     →   -O0 / -O1 / -O2
点击 Build               →   执行编译 → 链接 → 生成 hex 全流程
点击 Download            →   调用烧录工具写入芯片
点击 Debug / 断点         →   启动 GDB + 调试服务器
```

**一句话总结：** Keil 是一个把 **工具链 + 构建系统 + 烧录器 + 调试器** 打包在一起的 IDE，它用图形界面隐藏了所有命令行细节。离开它，你需要自己把这四块分别装好、配好、连起来。

