# Xiangshan Project Guide

## Xiangshan Project Links

- GitHub: [https://github.com/OpenXiangShan](https://github.com/OpenXiangShan)

- Code Cloud/Gitee: [https://gitee.com/OpenXiangShan](https://gitee.com/OpenXiangShan)

- GitLink: [https://gitlink.org.cn/OpenXiangShan/XiangShan](https://gitlink.org.cn/OpenXiangShan/XiangShan)

## Xiangshan Document Link

- Processor Core Introduction: [https://xiangshan-doc.readthedocs.io/zh_CN/latest/arch/](https://xiangshan-doc.readthedocs.io/zh_CN/latest/arch/)

- Agile Development Framework

- Instructions for Use: [https://xiangshan-doc.readthedocs.io/zh_CN/latest/tools/xsenv/](https://xiangshan-doc.readthedocs.io/zh_CN/latest/tools/xsenv/)

- Tool Introduction: [https://xiangshan-doc.readthedocs.io/zh_CN/latest/tools/difftest/](https://xiangshan-doc.readthedocs.io/zh_CN/latest/tools/difftest/)

- Troubleshooting: [https://xiangshan-doc.readthedocs.io/zh_CN/latest/tools/troubleshoot/](https://xiangshan-doc.readthedocs.io/zh_CN/latest/tools/troubleshoot/)

- Toolchain compilation: [https://xiangshan-doc.readthedocs.io/zh_CN/latest/compiler/gnu_toolchain/](https://xiangshan-doc.readthedocs.io/zh_CN/latest/compiler/gnu_toolchain/)

## Introduction to Xiangshan project repository

Xiangshan's own RTL code:

* [**XiangShan**](https://github.com/OpenXiangShan/XiangShan): Xiangshan processor implementation

* [**HuanCun**](https://github.com/OpenXiangShan/HuanCun): Xiangshan processor's non-blocking L2/L3 cache

* [**FuDian**](https://github.com/OpenXiangShan/fudian): Xiangshan processor's floating point unit

Xiangshan's simulation environment:

* [**Difftest**](https://github.com/OpenXiangShan/XiangShan): Processor co-simulation framework used by Xiangshan

* [**NEMU**](https://github.com/OpenXiangShan/NEMU/tree/master): A high-performance ISA with efficiency close to QEMU Interpreter, here is an [introduction video](https://www.bilibili.com/video/BV1Zb4y1k7RJ)

* [**nexus-am**](https://github.com/OpenXiangShan/nexus-am): Abstract Machine, provides the runtime environment of the program, here is a [simple introduction](https://nju-projectn.github.io/ics-pa-gitbook/ics2020/2.3.html)

* [**DRAMsim3**](https://github.com/OpenXiangShan/DRAMsim3): Cycle-level simulation of memory behavior, configured for Xiangshan project

* [**env-scripts**](https://github.com/OpenXiangShan/env-scripts): Some script files, including performance analysis, module replacement and timing analysis, etc.

* [**xs-env**](https://github.com/OpenXiangShan/xs-env): Xiangshan processor front-end development environment deployment script

Xiangshan documents:

* [**XiangShan-doc**](https://github.com/OpenXiangShan/XiangShan-doc): Xiangshan processor documentation, including design documents, public reports, and clarification of erroneous news

Other repositories under the Xiangshan project include **riscv-linux**, **riscv-pk**, **riscv-torture**, etc.

## Xiangshan processor directory structure

[**XiangShan**](https://github.com/OpenXiangShan/XiangShan) The directory structure of the main warehouse is as follows:

```
.
├── scripts # Generate some scripts for Verilog and simulation
├── src # Structural design and verification code
│   └── main # Structural design code
│      └── scala
│      ├── device # Some peripherals for simulation
│      ├── system # SoC description
│      ├── top # Top-level file
│      ├── utils # Some basic hardware tool libraries
│      ├── xiangshan # Xiangshan CPU part design code
│      └── xstransforms # Some FIRRTL Transforms
├── fudian # Xiangshan floating point submodule
├── huancun # Xiangshan L2/L3 cache submodule
├── difftest # Xiangshan co-simulation framework
├── ready-to-run # Pre-compiled nemu dynamic link library and some loads
└── rocket-chip # Used to obtain the Diplomacy framework (waiting for upstream split)
```

