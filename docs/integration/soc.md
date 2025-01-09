# System-on-Chip Integration

It is worth noting that these are typical configurations, and users can also specify different architectural parameters.

## Clock and Reset Clock and Reset

NANHU has two clock domains:<br>NANHU has two clock domains:

* One clock domain for the cpu core (`io_clock`). All clock in CPU core is `io_clock`.

* One clock domain for jtag (`io_systemjtag_jtag_TCK`).

NANHU has the following reset signals.
All Reset signals are Asynchronous Resets.

* `io_reset` is internally synced with the clock `io_clock`. Active high.

* `io_systemjtag_reset` is internally synced with the clock `io_systemjtag_jtag_TCK`. Active high.

There are two types of synchronizers in the RTL. All instances are used to synchronize the reset.

* `AsyncResetSynchronizerPrimitiveShiftReg` (synchronize reset from `jtag_clock` to `io_clock`, from low frequency to high frequency)

* `ResetGen/ResetGenDFT` (used to synchronize for both `jtag_clock` and `io_clock`, depending on the instantiations)

## AXI Interface AXI Bus Interface

For our definitions of the three ports,<br>Here are the definitions of the three AXI bus interfaces in South Lake.

- 256-bit AXI Memory Port: for CPU to access DDR (CPU is the master)

Address space: DDR space (0x8000_0000 to 0xf_ffff_ffff)

AXI ID width: 6-bit

Number of outstanding transactions: 56 (max. supported by L3 cache)

Max transaction size: 64 bytes

Number of IDs: 64

Max number of outstanding transactions per ID: 1

- 64-bit AXI Peripheral Port: for CPU to access peripherals (CPU is the master)<br>64-bit AXI peripheral port: for CPU to access peripherals (CPU is the master device)

Address space: peripheral slaves space (0x0000_0000 to 0x7fff_ffff)<br>Address space: peripheral slave space (0x0000_0000 to 0x7fff_ffff)

AXI ID width: 2-bit<br>AXI ID width: 2 bits

Number of outstanding: 1 for instruction and 1 for data for each core<br>Number of outstanding transactions: 1 instruction and 1 data for each core

Max transaction size: 8 bytes<br>Maximum transaction size: 8 bytes

Number of IDs: 4 (though the ID is 4-bit, most of them are not used)<br>ID number: 4 (although the ID is 4-bit, most of them are not used)

Max number of outstanding transactions per ID: 1<br>Maximum number of outstanding transactions per ID: 1

- 256-bit AXI DMA Port: for peripherals to access CPU/DDR (CPU is the slave)<br>256-bit AXI DMA Port: for peripherals to access CPU/DDR (CPU is the slave)

Address space: DDR space (0x8000_0000 to 0xf_ffff_ffff)<br>Address space: DDR space (0x8000_0000 to 0xf_ffff_ffff)

AXI ID width: 8-bit<br>AXI ID width: 8 bits

Number of outstanding supported by CPU: 32<br>Number of outstanding transactions supported by CPU: 32

Different transaction sizes and transactions per ID are supported. They are changed into 256-bit TileLink transactions internally. One AXI transaction may be split into multiple TileLink transactions.<br>The AXI interface supports transactions of different transaction sizes and per ID. They are converted internally into 256-bit TileLink transactions. One AXI transaction can be split into multiple TileLink transactions.

This DMA port allows the peripherals to access DDR under coherency with the CPU caches. They can send standard AXI aw/w/ar requests to this port to write/read the DDR. Memory coherency is supported in the DMA port by the L3 cache. No explicit action is needed to sync the data with CPU.<br>The DMA port allows the peripherals to access DDR under coherency with the CPU caches. They can send standard AXI aw/w/ar requests to this port to write/read the DDR. Memory coherency is supported in the DMA port by the L3 cache. No explicit action is needed to sync the data with CPU.

Note: RISC-V defines a weak memory order. Load and store instructions are not required to be visible to other harts (CPU cores, DMA devices, etc) before an explicit memory barrier instruction like `fence`. For example, in NANHU, data in store queue and committed store buffer is not under coherency with the
CPU caches. To read newly stored data through DMA AXI port, a fence instruction must be used in software to make the CPU stores visible to the coherent cache hierarchy. This is a RISC-V requirement, not a bug/issue of NANHU.<br> This is a requirement of RISC-V, not a bug/problem with Southlake.

## Memory Map &nbsp; Memory Map

There are three main regions:<br>There are three main regions in the memory map:

* Peripheral slaves space 0x0000_0000 to 0x7fff_ffff<br>Peripheral slave space (0x0000_0000 to 0x7fff_ffff)

* DDR space: 0x8000_0000 to 0xf_ffff_ffff<br>DDR space (0x8000_0000 to 0xf_ffff_ffff)

* DMA to access DDR: 0x8000_0000 to 0xf_ffff_ffff<br>DMA to access DDR (0x8000_0000 to 0xf_ffff_ffff)

Peripherals in the Peripheral address space is mostly determined by the SoC.<br>The peripherals in the peripheral address space are mostly determined by the SoC.

Default PMA is configured according to NANHU mamory map. Illegal memory access will cause access fault exceptions.

Please remember to update the default PMA settings if you need to change the peripheral address mappings.<br>Default PMA is configured according to NANHU memory map. Illegal memory access will cause access fault exceptions.

If you need to change the peripheral address mappings, please remember to update the default PMA settings.

Internal peripheral address space:<br>The following is the internal peripheral address space:

| Device | Start Address | End Address |
| ------- | ---------- | -------- |
| CLI
