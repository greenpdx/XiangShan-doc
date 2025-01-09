# CPU Core Overview 

XiangShan-2 (NANHU) supports single-core and dual-core configurations, where each core has its own private L1/L2 cache. L3 is shared by multiple cores.

NANHU communicates with the uncore through 3 AXI interfaces, including the memory port, the DMA port and the peripheral port. It also has clock, reset, and JTAG interfaces. Please refer to the integration guide for more detailed information.

NANHU targets 2GHz@14nm, and 2.4GHz~2.8GHz@7nm.

## Typical Configurations &nbsp; Typical configurations
Below is the typical NANHU core configurations:

| Feature | NANHU (XiangShan-2) |
| ------- | ------------------- |
| Pipeline stage  | 11 |
| Decoder width  | 6 |
| Rename width  | 6 |
| Reorder buffer | 256 |
| Physical register width | 192(integer), 192(float) |
| Load queue | 80 |
| Store queue | 64 |
| L1 instruction cache | 64KB/128KB (4/8-way) |
| L1 Data Cache | 64KB/128KB(4/8 way) |
| L2 Cache | 512KB/1MB, 8-way, non inclusive |
| L3 Cache | 2MB~8MB, 8-way, non inclusive |
| Physical register size | 192x64 bits, 14R8W |
| ECC Support | Y |
| Virtual Memory Support | Y |
| Physical memory protection | Y |
| Virtualization | N |
| Vectorization | N |

## ISA Support &nbsp; ISA Support

| Instruction Set | Description |
| ------- | ------------------- |
| I | Integer Instruction |
| M | Integer Multiplication and Division Instructions|
| A | Atomic Instructions|
| F | Single-Precision Floating-Point Instructions|
| D | Double-Precision Floating-Point Instructions|
| C | 16-bit Compressed Instructions|
| Zba | Bitmanip Extension - address generation instructions|
| Zbb | Bitmanip Extension - basic bit manipulation instructions|
| Zbc | Bitmanip Extension - carryless multiplication instructions|
| Zbs | Bitmanip Extension - single-bit instructions|
| zbkb | Cryptography Extension - Bitmanip instructions|
| Zbkc | Cryptography Extensions - Carryless Multiplication Instructions|
| zbkx | Cryptography Extensions - Crossbar permutation instructions|
| zknd | Cryptography Extensions - AES Decryption Instructions|
| zkne | Cryptography Extensions - AES Encryption Instructions|
| zknh | Cryptography Extensions - AES Extensions - Hash Function Instructions|
| zksed | Cryptography Extensions - SM4 Block Cipher Instructions|
| zksh | Cryptography Extensions - SM3 Hash Function Instructions|
| svinval | Fine-Grained Address Translation Cache Invalidation Instructions|

## Instruction Latency &nbsp; Instruction Latency

Most arithmetic instructions are single-cycle instructions (`Latency = 1`). 
Multi-cycle instructions are listed in the following table:

| Instruction(s) / Operations | Latency | Descriptions |
| -------------- | ------- | ------------ |
| `LD` | 4 (to ALU and LD), 5 (others) | Load operations (to use) |
| `MUL` | 3 | Integer multiplication |
| `DIV` | 4~20 | Integer division (SRT16) |
| `FMA` | 5 | Floating-point multiplication and addition instruction (cascade FMA) |
| `FADD`, `FMUL` | 3 | Floating-point addition/multiplication operations |
| `FDIV/SQRT` | 3~18 | Floating-point division/square root operations|
| `CLZ(W)`, `CTZ(W)`, `CPOP(W)`, `XPERM(8/4)`, `CLMUL(H/R)` | 3 | Complex bit operations|
| `AES64*`, `SHA256*`, `SHA512*`, `SM3*`, `SM4*` | 3 | Complex scalar crypto operations|

## Priviledge Mode &nbsp; Privilege Level

NANHU supports three levels of privilege mode: machine(M), supervisor(S), and user(U).

## Microarchitecture &nbsp; Microarchitecture

For more details, please refer to the CPU Core section

## Cache Controller &nbsp; Cache Controller

Nanhu's cache controller is connected to the L3 cache, which is used to perform cache maintenance operations (CMO). Programmers ought to use MMIO based memory access to trigger operation required.

The following is a register table of the L3 cache controller.

| Address | Width | Attr. | Description |
| ------- | ----- | ----- | ----------- |
| 0x3900_0100 | 8B | RW | `Tag register` of the interest cache block |
| 0x3900_0108 | 8B | RW | `Set register` of the interest cache block |
| 0x3900_0110 | 8B | RW | `Way` register of the interest cache block (deprecated)  |
| 0x3900_0118 - 0x3900_0150 | 64B in total | RW | `Data` register of the interest cache block (deprecated)
