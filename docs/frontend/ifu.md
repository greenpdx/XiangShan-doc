# Instruction Fetch Unit Documentation
![ifu](../figs/frontend/IFU.png)

This chapter describes the implementation of the Instruction Fetch Unit (IFU) of the Xiangshan processor. The pipeline of the instruction fetch unit is shown in the figure above.

The IFU of the Nanhu architecture adopts a 4-stage pipeline structure, which is greatly simplified compared to the design of the Yanqi Lake version IFU, thanks to the branch prediction-instruction cache decoupled instruction fetch architecture.

After an instruction fetch request is sent from the FTQ, it goes through the following stages in the IFU:

- The instruction fetch request sent from the FTQ contains a 32-byte instruction code (called an instruction block) with the starting address and the address of the next jump target,

- In the `IFU0` stage, the request is sent to the IFU pipeline and the ICache module at the same time.

- The `IF1` stage will do some simple calculations (for example, each 2 bytes in this instruction block, that is, the PC of each possible instruction).
- In the `IF2` stage, after the instruction cache returns data of up to two cache lines (because there may be a situation where the instruction block crosses lines), the first step is to split the instruction, discard the instruction code outside the instruction fetch address, and get the instruction code in the valid range. Send it to the predecoder for predecoding, and expand the 16-bit compressed instruction to a 32-bit instruction.
- In the `IF3` stage, the predecoding result will first be sent to the [branch prediction checker](#predchecker). If an error is found, the IFU pipeline will be refreshed in the next beat and the information will be sent to the FTQ to refresh the predictor and fetch the instruction again. The cache without error is waiting for decoding in the instruction buffer queue (IBuffer).
- In the `IF3` stage, according to the result of address translation, it will also initiate an instruction fetch request to the instruction MMIO module, and at the same time change to [MMIO instruction fetch mode](#mmiofetch), and the instructions will be executed one by one in sequence.
- The IFU control logic also needs to [handle half of the RVI instruction](#half).


## Predecoding

The predecoder decodes the 16 16-bit instruction codes after segmentation to obtain some instruction information (whether it is a jump instruction, the type of jump instruction, whether it is a compressed instruction, etc.), and calculates its target address for the jump instruction. This is mainly to provide the [branch prediction checker](#predchecker) with instruction information and the correct target address and to update the instruction information in the predictor in a timely manner.

On the other hand, the predecoder will also expand the compressed instruction (if there is one in this instruction block) to a 32-bit long instruction to simplify the decoding logic later.


## Branch prediction check

After getting the pre-decoding information of the instruction, the branch prediction checker mainly checks for the following errors:

- Errors in predicting that jump instructions do not jump: Check for the two instructions that must jump, `jal` and `ret`. If these two instructions are in the [valid instruction range](#validinstr) of this block and are predicted not to jump, it is considered a prediction error.
- Prediction errors of non-jump instructions: If the instruction predicted to jump is not in the valid instruction range, or is in the valid instruction range but is not a jump instruction, it is considered a prediction error.
- Target address error: For jump instructions whose target address can be known from the instruction code (except jump instructions other than `jalr`), if it is in the valid instruction range and is predicted to jump and the jump target address does not match the correct address, it is considered a prediction error.

After discovering an error, the branch prediction checker selects the most predictive error instruction in the previous instruction sequence, passes the error information (the position of the error instruction in the block, instruction pre-decoding information, and the correct target address) to the FTQ, and clears the IFU pipeline. The IFU waits for the FTQ to resend the instruction fetch request.


## Cross-line Fetch

Since our instruction block includes 32 bytes of instruction code, which is equivalent to the size of half a cache line (64 Bytes), if the starting address of this block is in the second half of the cache line, it is entirely possible that the range of the block spans two cache lines. Therefore, the instruction cache supports fetching two cache lines at a time to ensure instruction throughput in this case. The specific approach is that when the FTQ finds that the starting address of the block is in the second half of the cache line, it initiates a request for two adjacent cache lines of the instruction cache.


## Valid instruction range

The valid range of a fetch instruction block is determined by the starting address given by FTQ and the index of the jump instruction (if there is a jump). If there is no jump instruction in this block, the default instruction valid range is 256 bits starting from the starting address.

The valid instruction range may be re-determined by the IFU check, mainly including:

* When the branch prediction check finds unpredicted jumps of `jal` and `ret` instructions, the valid instruction range needs to be shortened to the first such jump instruction;

* When the previous block has half an RVI, the first 16 bits of the following block are not in the valid instruction range.
* The effective range of the instruction of the block requested by MMIO is only 32 bits


## MMIO instruction fetch

In the `IF3` stage, if ITLB finds that the address is in the MMIO space, IFU starts the MMIO instruction fetch mode and sends a request to the instruction MMIO module. The instruction MMIO module sends a `Get` request for 64 bits of data to the MMIO bus, waits for the bus to return, and then cuts the data according to the IFU's request address and returns the instruction code. IFU extends the instruction code and sends it to the instruction buffer queue. At the same time, IFU blocks the pipeline and listens to the commit signal of ROB until the instruction is executed and sends the front-end redirection to fetch the next instruction.

MMIO requests only fetch one instruction at a time, so the instruction execution speed of the processor will become very slow in this mode.


## Handling of half RVI instruction

When an instruction block finds that its last 2 bytes are the first half of an RVI instruction in the `IF3` stage, we count this RVI instruction in this block. At the same time, our mechanism of fetching two cache lines ensures that the second half can be fetched. Therefore, we only need to set a flag when this happens, and exclude the first 2 bytes from the valid range of the instruction when the next block comes.

--8<-- "docs/frontend/abbreviations.md"
