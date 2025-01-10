# Custom L1 cache operation

Nanhu architecture supports custom L1 cache operation based on CSR.

## L1 Cache instruction register

The cache instruction register is divided into 3 categories: a `CSR_CACHE_OP` cache instruction register, a `CSR_OP_FINISH` cache instruction status register, `CSR_CACHE_*`: L1 cache control register, the parameter configuration and result return of custom cache instructions are all performed through these registers. The base address of the register is specified by `Sfetchctl`, which defaults to `Sfetchctl`. The detailed register list is as follows in order:

Register name|Description
-|-
CSR_CACHE_OP|cache op instruction code. Writing to this register will trigger the execution of custom L1 cache instructions
CSR_OP_FINISH|L1 cache instruction completion flag
CACHE_LEVEL|cache instruction target selection. 0: ICache, 1: DCache
CACHE_WAY|cache way select
CACHE_IDX|cache index select
CACHE_BANK_NUM|cache bank select
CACHE_TAG_ECC|tag ecc
reserved|reserved
CACHE_TAG_LOW|tag data
CACHE_TAG_HIGH|reserved
reserved|reserved
CACHE_DATA_ECC|data ecc
CACHE_DATA_X|data [64(X+1)-1:64(X)]

<!-- TODO: Streamline the encoding space -->

## cache instruction code

The opcodes supported by custom L1 cache instructions are as follows:

Operation|Opcode
-|-
READ_TAG_ECC|0
READ_DATA_ECC|1
READ_TAG|2
READ_DATA|3
WRITE_TAG_ECC|4
WRITE_DATA_ECC|5
WRITE_TAG|6
WRITE_DATA|7
<!-- COP_FLUSH_BLOCK|8 -->
## Custom L1 cache instruction basic execution flow

1. Use CSR instruction to write the parameter configuration register in the cache instruction register and clear the OP_FINISH register
1. Write the instruction code to the CSR_CACHE_OP register
1. (Optional) Poll CSR_OP_FINISH until the instruction is completed and CSR_OP_FINISH == 1
1. Use CSR instruction to read the result register in the cache instruction register to obtain the cache instruction result
