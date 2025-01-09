# Fetch Target Queue (FTQ)
This chapter describes the implementation of the Fetch Target Queue (FTQ) of the Xiangshan processor.
<figure markdown>
![ftq](../figs/frontend/ftq.svg){ width="700" }
<figcaption>Interaction between FTQ and other modules</figcaption>
</figure>

## Function description
FTQ is a buffer queue between the branch prediction and instruction fetch units. Its main function is to **temporarily store the instruction fetch targets predicted by the BPU** and **send instruction fetch requests to the IFU based on these instruction fetch targets**. Another important function is to **temporarily store the prediction information of each predictor of the BPU** and send this information back to the BPU for predictor training after the instruction is submitted. Therefore, it needs to **maintain the complete life cycle of the instruction from prediction to submission**. Since the backend has a large overhead for storing PC, when the backend needs the instruction PC, it will go to the FTQ to read it.

## Internal structure
FTQ is a queue structure, but the content of each item in the queue is stored in a different storage structure according to its own characteristics. These storage structures mainly include the following:

- **ftq_pc_mem**: register stack implementation, storing information related to instruction addresses, including the following fields

- `startAddr` predicts the starting address of the block

- `nextLineAddr` predicts the starting address of the next cache line of the block

- `isNextMask` predicts whether the starting position of each possible instruction in the block is in the next area aligned according to the predicted width

- `fallThruError` predicts whether the next sequential instruction fetch address is wrong

- **ftq_pd_mem**: register stack implementation, storing the decoding information of each instruction in the predicted block returned by the instruction fetch unit, including the following fields

- `brMask` whether each instruction is a conditional branch instruction

- `jmpInfo` predicts the information of the unconditional jump instruction at the end of the block, including whether it exists, whether it is `jal` or `jalr`, whether it is a `call` or `ret` instruction

- `jmpOffset` predicts the location of the unconditional jump instruction at the end of the block

- `jalTarget` predicts the end of the block `jal` jump address
- `rvcMask` whether each instruction is a compressed instruction
- **ftq_redirect_sram**: SRAM implementation, stores the prediction information that needs to be restored during redirection, mainly including information related to RAS and branch history

- **ftq_meta_1r_sram**: SRAM implementation, stores the rest of the BPU prediction information

- **ftb_entry_mem**: register stack implementation, stores the necessary information of the FTB item during prediction, used to train new FTB items after submission

In addition, some information such as queue pointers and the status of each item in the queue is implemented using registers.

## Life cycle of instructions in FTQ
Instructions are sent to FTQ after being predicted by BPU in units of [prediction blocks](./bp.md#pred-block). FTQ will not completely release the items corresponding to the [prediction block](./bp.md#pred-block) in the storage structure until all instructions in the [prediction block](./bp.md#pred-block) where the instruction is located are submitted to the backend. What happens in this process is as follows:

1. The prediction block is sent from the BPU and enters the FTQ. The `bpuPtr` pointer is incremented by one, the various states of the corresponding FTQ items are initialized, and various prediction information is written into the storage structure; if the prediction block comes from the BPU overwrite prediction logic, `bpuPtr` and `ifuPtr` are restored

2. FTQ sends an instruction fetch request to the IFU, the `ifuPtr` pointer is incremented by one, and waits for the pre-decoding information to be written back

3. The IFU writes back the pre-decoding information, the `ifuWbPtr` pointer is incremented by one, if the pre-decoding detects a prediction error, the corresponding redirection request is sent to the BPU, and `bpuPtr` and `ifuPtr` are restored

4. The instruction enters the backend for execution. If the backend detects a misprediction, the FTQ is notified, a redirection request is sent to the IFU and BPU, and `bpuPtr`, `ifuPtr` and `ifuWbPtr` are restored

5. The instruction is submitted at the backend and notified FTQ, when all valid instructions in the FTQ item have been submitted, the `commPtr` pointer is incremented by one, the corresponding information is read from the storage structure, and sent to the BPU for training

The life cycle of the instructions in the prediction block `n` will involve the four pointers `bpuPtr`, `ifuPtr`, `ifuWbPtr` and `commPtr` in FTQ. When `bpuPtr` starts to point to `n+1`, the instructions in the prediction block enter the life cycle. When `commPtr` points to `n+1`, the instructions in the prediction block complete the life cycle.

## Other functions of FTQ
- Since BPU is basically non-blocking, it can often get ahead of IFU, so the instruction fetch requests provided by BPU that have not been sent to IFU can be used for instruction prefetching. This part of logic is implemented in FTQ, and prefetch requests are sent directly to the instruction cache
- FTQ summarizes most of the information of the front end, so it implements many performance counters that can be obtained during simulation. For details, see [source code](https://github.com/OpenXiangShan/XiangShan/blob/20bb5c4c094f06264df0e406d0df058f04ccc21c/src/main/scala/xiangshan/frontend/NewFtq.scala#L1024-L1206)

-- 8 <-- "docs/frontend/abbreviations.md"
