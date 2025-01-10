# Decode Unit
## Basic Function
Instructions are taken from the instruction cache, sent to the instruction buffer (queue) for temporary storage, and then sent to the decode unit for decoding at a rate of 6 per cycle, and then passed to the next pipeline stage.

## Instruction Fusion
The decode unit supports the fusion of some instructions. In the `FusionDecoder` module, instruction fusion is completed based on the 32-bit data of two consecutive instructions. For two non-consecutive instructions, Xiangshan currently does not support instruction fusion.

Currently, Xiangshan supports the following instruction fusion:

* clear upper 32 bits / get lower 32 bits: `slli r1, r0, 32` + `srli r1, r1, 32`
* clear upper 48 bits / get lower 16 bits: `slli r1, r0, 48` + `srli r1, r1, 48`
* clear upper 48 bits / get lower 16 bits: `slliw r1, r0, 16` + `srliw r1, r1, 16`
* sign-extend a 16-bit number: `slliw r1, r0, 16` + `sraiw r1, r1, 16`
* shift left by one and add: `slli r1, r0, 1` + `add r1, r1, r2`
* shift left by two and add: `slli r1, r0, 2` + `add r1, r1, r2`
* shift left by three and add: `slli r1, r0, 3` + `add r1, r1, r2`
* shift zero-extended word left by one: `slli r1, r0, 32` + `srli r1, r0, 31`
* shift zero-extended word left by two: `slli r1, r0, 32` + `srli r1, r0, 30`
* shift zero-extended word left by three: `slli r1, r0, 32` + `srli r1, r0, 29`
* get the second byte: `srli r1, r0, 8` + `andi r1, r1, 255`
* shift left by four and add: `slli r1, r0, 4` + `add r1, r1, r2`
* shift right by 29 and add: `srli r1, r0, 29` + `add r1, r1, r2`
* shift right by 30 and add: `srli r1, r0, 30` + `add r1, r1, r2`
* shift right by 31 and add: `srli r1, r0, 31` + `add r1, r1, r2`
* shift right by 32 and add: `srli r1, r0, 32` + `add r1, r1, r2`
* add one if odd, otherwise unchanged: `andi r1, r0, 1` + `add r1, r1, r2`
* add one if odd (in word format), otherwise unchanged: `andi r1, r0, 1` + `addw r1, r1, r2`
* `addw` and extract its lower 8 bits (fused into `addwbyte`)
* `addw` and extract its lower 1 bit (fused into `addwbit`)
* `addw` and `zext.h` (fused into `addwzexth`)
* `addw` and `sext.h` (fused into `addwsexth`)
* logic operation and extract its LSB
* logic operation and extract its lower 16 bits
* `OR(Cat(src1(63, 8), 0.U(8.W)), src2)`
* mul 7-bit data with 32-bit data
