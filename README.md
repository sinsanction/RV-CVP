# RV-CVP

RV-CVP 开源项目，包括以下三个部分：
1. [NutShell with RV-CVP extension](https://github.com/sinsanction/NutShell)，带有 RV-CVP 指令集扩展的果壳处理器硬件设计。
2. [RV-CVP CNNAPI](https://github.com/sinsanction/rv-cvp-cnnapi)，RV-CVP 指令集软件编程框架（CNNAPI 编程库）。
3. [RV-CVP Test](https://github.com/sinsanction/rv-cvp-test)，RV-CVP 指令集测试程序。

## 在仿真环境中运行

仿真运行需要两部分内容，一是由 NutShell 硬件设计生成模拟器，二是编译生成在模拟器上运行的程序。

前者通过 mill 和 Verilator 实现，后者通过 AM ——一个裸机运行时环境来编译生成可在模拟器上运行的程序。

### 1.克隆以下仓库

NutShell: https://github.com/sinsanction/NutShell

AM: https://github.com/sinsanction/nexus-am

xs-env: https://github.com/OpenXiangShan/xs-env

其中 NutShell 需要克隆其子模块 difftest，用于生成模拟器。而 xs-env 只需要用到其中构建环境的脚本，不必克隆其中的子模块。

### 2.生成模拟器

使用 `xs-env` 中的 `setup-tools.sh` 脚本，安装 mill、Verilator 和相关依赖。

切换到 `NutShell` 目录，确保子模块 `difftest` 已经存在。

运行 `env.sh`，设置环境变量。
```
source ./env.sh
```

生成模拟器 emu。
```
make emu EMU_CXX_EXTRA_FLAGS="-DFIRST_INST_ADDRESS=0x80000000"
```

### 3.编译测试程序

目前仿真环境支持的测试程序需要在 AM 下编写，只支持 C/C++ 程序。

#### 从头开始在 AM 创建一个程序

见 [https://github.com/OSCPU/nexus-am](https://github.com/OSCPU/nexus-am)。注意该仓库已不再维护，只看教程即可，实际使用请选择上面的 AM。

#### 使用现有程序

我们提供了一些预先写好的 RV-CVP 指令集程序，已经集成在 AM 中。

[cnnbench](https://github.com/sinsanction/nexus-am/tree/master/apps/cnnbench): 单个算子的测试程序。

[cnnapibench](https://github.com/sinsanction/nexus-am/tree/master/apps/cnnapibench): CNNAPI 框架以及真实 LeNet-5 模型的测试程序。

#### 编译方法

将 `AM_HOME` 设为 AM 的绝对路径，可以使用 `env.sh` 来设置。

以编译 cnnbench 中的功能测试程序为例。

```
cd nexus-am/apps/cnnbench
make ARCH=riscv64-nutshell mainargs=testfunc
```

最后在 `nexus-am/apps/cnnbench/build` 下会生成 bin 文件。

### 4.在模拟器上运行

```
path/to/emu --no-diff -b 0 -e 0 -C 10000000 -i path/to/*.bin
```
 - `--no-diff`: 必须要指定，因为目前没有实现自定义 RV-CVP 指令的差分对比。
 - `-b`: 打印 debug 记录的起始周期。一般只在查找硬件 bug 时启用。
 - `-e`: 打印 debug 记录的结束周期。
 - `-C`: 仿真执行的最大周期数，超过后强制停止。如果程序没有死循环，仿真将在执行完毕后自动退出，一般不必设置该项。
 - `-i`: 输入的测试程序镜像。

仿真运行结束后，模拟器会打印运行所用的周期数、指令数等信息。


## 在 FPGA 上运行

在 FPGA 上运行需要三部分内容：一是对应板卡的 Vivado 工程，二是由 NutShell 硬件设计生成 Verilog 文件，三是生成板卡上可运行的测试程序。

### 1.构建 Vivado 工程

从头开始构建 Vivado 较复杂，详细见 NutShell 的 [Run on FPGA](https://github.com/sinsanction/NutShell#run-on-fpga) 部分。

这里提供一个建好的，针对 Alveo U250 FPGA 板卡的工程，[https://github.com/ssdfghhhhhhh/NutShell_U250](https://github.com/ssdfghhhhhhh/NutShell_U250)。

### 2.编译生成 Verilog

确保已安装 `mill`，安装方法见上文。

修改 `Makefile` 文件14行

```
DATAWIDTH ?= 64
BOARD ?= pynq  # sim  pynq  axu3cg
CORE  ?= inorder  # inorder  ooo  embedded
```

修改 `NutShell\src\main\scala\top\Settings.scala` 47行

```
object PynqSettings {
  def apply() = Map(
    "FPGAPlatform" -> true,
    "NrExtIntr" -> 3,
    "ResetVector" -> 0x80000000L,
    "MemMapBase" -> 0x0000000080000000L,
    "MemMapRegionBits" -> 28,
    "MMIOBase" -> 0x0000000040600000L,
    "MMIOSize" -> 0x0000000001000000L
  )
}
```

在果壳根目录下执行 `make`，在 `NutShell\build` 下将生成 Verilog 代码：`TopMain.v`。

### 3.编译生成板卡上可运行的测试程序

请查看：[https://github.com/ssdfghhhhhhh/NutShell_U250/blob/main/compile_bbl.pdf](https://github.com/ssdfghhhhhhh/NutShell_U250/blob/main/compile_bbl.pdf)

### 4.连接 FPGA 板卡运行

请查看：[https://github.com/ssdfghhhhhhh/NutShell_U250/blob/main/README.pdf](https://github.com/ssdfghhhhhhh/NutShell_U250/blob/main/README.pdf)。

其中生成完比特流之后，可以查看综合与实现的报告。
