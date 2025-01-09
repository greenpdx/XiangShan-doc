# Backend pipeline

## Overall design

The pipeline backend of the processor is responsible for renaming and out-of-order execution of instructions. As shown in the figure below, the backend of the Xiangshan processor can be divided into 4 parts: CtrlBlock, IntBlock, FloatBlock, and Memblock. CtrlBlock is responsible for decoding, renaming, and dispatching instructions. IntBlock, FloatBlock, and MemBlock are responsible for out-of-order execution of integer, floating point, and memory access instructions respectively.

<figure markdown>
![bpu](../figs/backend/backend.svg){ width="800" }
<figcaption>Backend structure diagram </figcaption>
</figure>

In these modules, there are many configurable parameters. In the current code, the above four blocks are configured as follows by default:

* CtrlBlock

* Decode/Rename/Dispatch Width = 6

* Read register stack before launch

* IntBlock

* 192 physical registers

* 4 * ALU + 2 * MUL/DIV + 1 * CSR/JMP

* FloatBlock

* 192 physical registers

* 4 * FMAC + 2 * FMISC

* MemBlock

* 2 * LOAD + 2 * STORE (where STORE is divided into data and address for independent calculation)

## Code Implementation

Currently, the Xiangshan backend code is written in a rough way, and the design is configurable through manual parameterization transfer. In the future, we plan to use Diplomacy to rewrite the code to achieve a clearer parameterized design.

In the code version of Xiangshan Nanhu, the reservation station, functional unit, and register stack are packaged into ExuBlock, which includes Scheduler and FUBlock. The former again includes the reservation station, register stack, and BusyTable, while the latter includes the corresponding functional unit. These levels of packaging are only for parameter passing and wiring, and there is very little effective logic code.

In the fixed-point ExuBlock, it includes (1) reservation stations for ALU, MUL/DIV, and MISC; (2) reservation stations for LOAD, STORE DATA, and STORE ADDR; (3) functional units of ALU, MUL/DIV, and MISC.

In the floating-point ExuBlock, it includes (1) reservation stations for FMA and FMISC; (2) functional units of FMA and FMISC.

In the memory access MemBlock, the design of the memory access subsystem is included.
