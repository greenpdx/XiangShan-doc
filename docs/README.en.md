# Xiangshan Project Introduction and Guide

## 1. What is Xiangshan?

Xiangshan is an open source, high-performance RISC-V processor, based on the Chisel hardware design language, supporting the RV64GC instruction set. During the development of the Xiangshan processor, a large number of open source tools including Chisel, Verilator, GTKwave, etc. were used to implement basic tools for processor development such as differential verification, simulation snapshots, and RISC-V checkpoints, and a set of processor agile development processes based on all open source tools including design, implementation, and verification were established.

**Xiangshan Project Link**：

- GitHub：[https://github.com/OpenXiangShan](https://github.com/OpenXiangShan)

- 码云：[https://gitee.com/OpenXiangShan](https://gitee.com/OpenXiangShan)


**Xiangshan Project Warehouse Introduction**：

[**XiangShan**](https://github.com/OpenXiangShan/XiangShan)：Xiangshan processor implementation

[**XiangShan-doc**](https://github.com/OpenXiangShan/XiangShan-doc)：Xiangshan processor documents, including design documents, public reports, and clarifications of erroneous news

[**NEMU**](https://github.com/OpenXiangShan/NEMU/tree/master)：A high performance ISA interpreter with efficiency close to QEMU, here is one[Introduction Video](https://www.bilibili.com/video/BV1Zb4y1k7RJ)

[**nexus-am**](https://github.com/OpenXiangShan/nexus-am)：Abstract Machine, which provides the runtime environment for programs. There is one[Brief Introduction](https://nju-projectn.github.io/ics-pa-gitbook/ics2020/2.3.html)

[**DRAMsim3**](https://github.com/OpenXiangShan/DRAMsim3)：cSimulate memory behavior at the cycle-level

[**env-scripts**](https://github.com/OpenXiangShan/env-scripts)：Some script files, including performance analysis, module replacement and timing analysis, etc.

And other repositories include **riscv-linux** , **riscv-pk** , **riscv-torture** and so on.

## 2. Xiangshan Processor Directory Structure

```
.
├── debug                # Some common commands for running tests are written in the form of scripts
├── scripts              # Generate Verilog and some scripts for simulation
├── src                  # Structural design and verification code
│   ├── main               # Structural design code
│   │   └── scala
│   │       ├── bus            # Some bus tools
│   │       ├── device         # Some peripherals for simulation
│   │       ├── difftest       # Differential Test Interface Description
│   │       ├── system         # Description of SoC
│   │       ├── top            # Top-level file
│   │       ├── utils          # Some basic hardware tool libraries
│   │       ├── xiangshan      # The design code of the Xiangshan CPU part
│   │       └── xstransforms   # Some FIRRTL Transforms
│   └── test               # Verification Code
│       ├── csrc             # C/C++ code for simulation
│       │   ├── common         # Common components in simulation
│       │   ├── difftest       # C/C++ interface implementation of differential test
│       │   ├── vcs            # Files used during VCS simulation
│       │   └── verilator      # Files used by Verilator simulation
│       ├── scala            # Some unit tests based on ChiselTest
│       │   ├── cache          # Cache Unit Test
│       │   ├── top
│       │   └── xiangshan      # Xiangshan core internal module unit test
│       ├── testcase
│       │   ├── Makefile
│       │   └── tests
│       └── vsrc             # Verilog code for simulation
│           ├── common
│           └── vcs
├── ready-to-run         # Precompiled nemu dynamic link library, and some loads
├── rocket-chip          # Used to get the Diplomacy framework (pending upstream split)
├── berkeley-hardfloat   # Modified hardfloat, currently used as Xiangshan's FPU
├── block-inclusivecache # The modified SiFive InclusiveCache is currently used as L2 and L3 of Xiangshan
└── chiseltest           # ChiselTest framework source code
```

## 3. Xiangshan processor architecture design 
The Xiangshan processor is designed with an out-of-order six-issue structure. The front end includes instruction fetch unit, branch prediction unit, instruction buffer and other units, and fetches instructions sequentially. The back end includes decoding, renaming, reordering buffer, reservation station, integer/floating-point register file, integer/floating-point operation unit. We separate the memory access subsystem, including two load pipelines and two store pipelines, as well as independent load queues and store queues, Store Buffer, etc. The cache includes modules such as I\$, D\$, L1+\$, L2\$, TLB and prefetcher. The position of each part in the pipeline and the parameter configuration can be obtained from the figure below.

![Xiangshan processor structure diagram](images/xs-arch-simple.svg)

The specific structural design of Xiangshan processor is introduced later.

## 4. Unit Tests

We implemented the Agent-Faker Cache test framework based on ChiselTest. Its design concept is similar to UVM and is currently applied to Dcache, L2Cache and TLB.

![Agent-Faker](images/AgentFaker.png)

## 5. Simulation test

The Xiangshan processor is mainly simulated using Verilator, and peripherals such as UART and SD card are simulated. During simulation, a runtime comparison is made with the simulator NEMU, and information such as the processor's clock interrupt is passed to the simulator, guiding the simulator to be consistent with the processor in key choices, thereby improving the flexibility of the simulation.

![Diff-test](images/difftest.png)

When the processor and simulator behaviors are inconsistent, the simulation program will stop and error analysis can be performed by viewing waveforms and logs. We developed Wave Terminator to extract waveforms with low-level semantics into logs with high-level semantics. In addition, we also developed the log viewer [LogViewer](https://github.com/OpenXiangShan/env-scripts/tree/main/logviewer) to make it easier to view logs.

## 6. Performance Evaluation

During simulation, each module can freely print counters, and you can choose to print counters in real time or at the end of simulation. Performance counters are analyzed through [script tools](https://github.com/OpenXiangShan/env-scripts/blob/main/perf/perf.py), and we have also developed visualization tools to analyze performance counters.

In order to test the performance of SPEC benchmarks and perform performance analysis on SPEC programs, NEMU is used to generate SimPoint of SPEC programs, conduct large-scale parallel testing, and quickly obtain SPEC scores. In addition, SPEC fragments can also be used for performance evaluation during development.
