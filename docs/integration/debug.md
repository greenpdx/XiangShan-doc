# JTAG and Debug

NANHU implements [RISC-V External Debug Support Version 0.13.2](https://riscv.org/wp-content/uploads/2019/03/riscv-debug-release.pdf).
NANHU uses components including Debug Module, Debug Module Interface, Debug Transport Module and JTAG Interface.

NANHU Debug Module supports Program Buffer (16 bytes) and System Bus Access. Abstract Commands are implemented using Program Buffer.
It is connected to L3 crossbar using TileLink to support System Bus Access.
It has two parts: DMInner and DMOuter. DMInner is driven by core clock while DMOuter is driven by JTAG clock. DMOuter issues debug interrupts.

## Debug &nbsp; 调试
NANHU's CSR indicates whether it is in Debug Mode through a single bit register.

Debug mode can be entered when:

* Debug Module issues a debug interrupt.

* An ebreak is executed in mode X and ebreakX is executed (X is M, S, or U).

* The hart has returned from Debug Mode, and step bit in dcsr is set. The hart enters debug mode after exactly one instruction has committed.
* When the trigger hits, the corresponding action bit in the mcontrol CSR is set to 1

NANHU implements the following debug mode registers:
Debug control and status register (dssr), Debug PC (dpc), Debug Scratch (dscratch and dscratch1).

### Debug control and status register Debug control and status register

NANHU implements optional dcsr bits ebreaks, ebreaku, mprven and step.
See [RISC-V External Debug Support Version 0.13.2](https://riscv.org/wp-content/uploads/2019/03/riscv-debug-release.pdf) for more detail.

### Debug PC &nbsp; Debug PC

Debug PC is used to store the program counter of the exact instruction that entered debug mode. PC is recovered when a dret instruction is executed. Debug Module can modify this CSR to manipulate the path of program execution.

### Debug Scratch &nbsp; Debug Scratch Register

There are 2 debug scratch registers (dscratch and dscratch1) in NANHU. These registers are used as scratch registers by the Debug Module.

## Trigger &nbsp; Trigger

### Trigger select register &nbsp; Trigger select register

Tselect selects one of the 10 triggers in NANHU. All NANHU's triggers are listed below.

| No. | Type | Timing bit | Chain bit | Select bit | Match bit | 
| --- | --- | --- | --- | --- | --- | 
| 0 | execute/store/load | Wired to 0 | Y | Y* | 0, 2 or 3 | 
| 1 | execute/store/load | | Y | Y* | 
| 2 | execute/store/load | | Y | Y* | 
| 3 | execute/store/load | | Y | Y* | 
| 4 | execute/store/load | | Y | Y* | 
| 5 | execute/store/load | | Y | Y* | 
| 6 | execute/store/load | | Y | Y* | 
| 7 | execute/store/load | | Y | Y* | 
| 8 | execute/store/load | | Y | Y* | 
| 9 | execute/store/load | | Wired to 0 | Y* | *: 

If the type is store or load, the select bit will be set to 0 for WARL.

### Trigger data status register &nbsp; Trigger data status register

All triggers in NANHU are match control triggers. Each trigger has a mcontrol (tdata1), data (tdata2) and info (tinfo) register.

## Breakpoint &nbsp; Breakpoint

NANHU supports using ebreak instruction as software breakpoint. To use this in privileged mode X, ebreakX bit in dcsr should be set. Hardware breakpoints are implemented using triggers.

## Debug memory map &nbsp; Debug memory map

The debug module uses address space from 0x3802_0000 to 0x3802_0fff.

## Debug Module Interface &nbsp; Debug module interface

DMI has a bus width of 7 bits.

## JTAG Interface &nbsp; JTAG interface

Nanhu’s JTAG interface supports asynchronous reset TRSTn.

## Connecting to Debug Module &nbsp; Connecting to debug module
First, you need to compile [riscv-openocd](https://github.com/riscv/riscv-openocd). For how to compile riscv-openocd, please refer to [Github README](https://github.com/riscv/riscv-openocd/blob/riscv/README).

Run simv with options: `./difftest/simv [+workload=WorkloadName.bin] +no-difftest
