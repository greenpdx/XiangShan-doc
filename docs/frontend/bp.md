# Branch Prediction
<figure markdown>
![bpu](../figs/frontend/bpu.svg){ width="800" }
<figcaption>BPU pipeline diagram</figcaption>
</figure>

This chapter describes the overall architecture of the Xiangshan processor branch prediction unit, and its prediction pipeline is shown in the figure above.

<!-- The Nanhu architecture adopts an instruction fetch architecture that decouples branch prediction and instruction cache. The branch prediction unit provides instruction fetch requests and writes them into a queue, which sends them to the instruction fetch unit and into the instruction cache. -->
The branch prediction unit adopts a multi-level hybrid prediction architecture, whose main components include [Next Line Predictor](#nlp) (Next Line Predictor, hereinafter referred to as NLP) and [Accurate Predictor](#apd) (Accurate Predictor, hereinafter referred to as APD). Among them, [NLP](#nlp) is a [uBTB](#ubtb) (micro BTB), [APD](#apd) consists of [FTB](#ftb)[^ftbcite], [TAGE-SC](#tage-sc), [ITTAGE](#ittage), [RAS](#ras). NLP provides bubble-free predictions, and the prediction results can be obtained in the next beat of the prediction request. The delay of each component of APD is between 2 and 3 beats. Among them, the delay of FTB, TAGE, and RAS is 2 beats; the delay of SC and ITTAGE is 3 beats. A prediction will go through three pipeline stages, and each pipeline stage will generate new prediction content. These predictors distributed in different pipeline stages form an [overriding predictor](#overriding-predictor) (overriding predictor).

In addition to whether it is decoupled from the instruction fetch unit, the biggest difference between the branch predictor of the Nanhu architecture and the previous generation (Yanqi Lake) architecture is the definition of the [prediction block](#pred-block). In the Nanhu architecture, the BTB is replaced by the FTB (Fetch Target Buffer). Each FTB item forms a prediction block. We not only predict the start address of the next block, but also the end point of the current block. See the [FTB](#ftb) section for details.

## Top-level module (BPU)
BPU (Branch Prediction Unit) is the top-level module of the branch predictor. It contains the coverage prediction logic and pipeline handshake logic, as well as the management of the global branch history.

### Handshake logic
Each pipeline stage of the BPU is connected to [FTQ](ftq.md). Once the first prediction pipeline stage has a valid prediction result, or the subsequent prediction pipeline stage produces a different prediction result, the handshake signal valid bit with [FTQ](ftq.md) will be set high.

### Global branch history management
The Southlake architecture implements a nearly completely accurate [global branch history](#global-history), which is guaranteed by the following three points:

- **Speculation update**: Each prediction will calculate a new global history based on the number of conditional branch instructions and the prediction direction in the [prediction block](#pred-block), and use it in the new prediction

- **Compare global history in the overwrite logic**: Once the global history after the speculation update of the subsequent pipeline stage is different from the result of the previous pipeline stage (the number of conditional branches or the execution result is different), the pipeline will also be flushed and the prediction will be restarted

- **Store a copy of the global history after prediction**: After the prediction is completed, the global history used in the current prediction will be stored in [FTQ](ftq.md), and read out and sent back to the BPU when the misprediction is recovered

The reason why it is "close to" completely accurate is that the BPU will ignore those conditional branch instructions that have never jumped. They will not be recorded in the FTB and will not be included in the branch history.

## Next Line Predictor (NLP)
The next line predictor aims to provide a bubble-free fast prediction stream with low storage overhead. Its functionality is mainly provided by [uBTB](#ubtb). For a given starting address PC, uBTB makes an overall prediction for a [prediction block](#pred-block) starting from PC.

### uBTB
It uses the branch history and the low-order XOR of PC to index the storage table, and directly provides the most concise prediction from the table, including the starting address of the next prediction block `nextAddr`, whether the prediction block has a branch instruction jump `taken`, and (if so) the offset of the jump instruction relative to the starting address `cfiOffset`. In addition, it provides relevant information about the branch instruction to update the branch history, including whether it is a conditional branch jump `takenOnBr`, and the number of branch instructions in the block `brNumOH`.

We have abandoned the tag matching approach, which will bring a problem. In the case of tag matching, if a [prediction block](#pred-block) does not hit, the PC of the next prediction block will be set to the current PC plus the [prediction width](#pred-width). To avoid waste, if there is no branch instruction in the prediction block, the uBTB will not be written during training. Under this premise, if there is no tag matching mechanism, it is easy to predict the prediction block without branch instructions (which does not hit under the tag matching mechanism) as another jump block. For this situation, we introduced another prediction mechanism to predict whether there may be a valid branch instruction in the current PC. The storage structure of this prediction mechanism is directly indexed by the fetch PC, and its query result indicates whether the fetch PC has been written. When it indicates that the PC has not been written, the next PC will be predicted as the current PC plus the prediction width.

## Accurate predictor (APD)
In order to improve the overall prediction accuracy and reduce pipeline flushing caused by prediction errors, the Nanhu architecture implements a prediction mechanism with higher latency and more accuracy.

The precise predictor includes the instruction target buffer [FTB](#ftb), the conditional branch direction predictor [TAGE-SC](#tage-sc), the indirect jump predictor [ITTAGE](#ittage) and the return address stack [RAS](#ras).

### FTB
FTB is the core of APD. The predictions made by other prediction components of APD all rely on the information provided by FTB. In addition to providing information about branch instructions in the [prediction block](#pred-block), FTB also provides the end address of the prediction block. For FTB, the generation strategy of FTB items is crucial. Based on the original paper [^ftbcite], the Nanhu architecture combines the ideas of this paper [^ftbcitequalcomm] to form the existing strategy. The starting address of the FTB item is *`start`* and the ending address is *`end`*. The specific strategy is as follows:

- **The FTB item is indexed by *`start`*, and *`start`* is generated in the prediction pipeline. In fact, *`start`* basically follows one of the following principles:**
- *`start`* is the *`end`* of the previous prediction block
- *`start`* is the target address of the redirection from outside the BPU;
- **At most two branch instructions are recorded in the FTB item, and the first one must be a conditional branch;**
!!! note inline end "Note"
Under this training strategy, the same branch instruction may exist in multiple FTB items.
- ***end* must satisfy one of the following three conditions:**
- *`end`* - *`start`* = prediction width
- *`end`* is the PC of the third branch instruction within the prediction width starting from *`start`*
- *`end`* is the PC of the next instruction of an unconditional jump branch, and it is within the prediction width starting from *`start`*

As in the implementation in the paper[^ftbcite], we only store the low bits of the end address, and the high bits are concatenated with the high bits of the start address. Similar to AMD[^amd], we also record the "always jump" bit for the conditional branch instructions in the [FTB](#ftb) item. This bit is set to 1 when the conditional branch jump is encountered for the first time. When it is 1, the direction of the conditional branch is always predicted to be a jump, and its result is not used to train the conditional branch direction predictor; when the conditional branch encounters a non-jump result, this bit is set to 0, and its direction is then predicted by the conditional branch direction predictor.

### TAGE-SC
TAGE-SC is the main predictor of conditional branches in the Nanhu architecture. Its general logic is inherited from TAGE-SC-L of the previous generation Yanqi Lake architecture. In the current implementation, the delay of TAGE is 2 beats and the delay of SC is 3 beats.
!!! note "Why is there no loop predictor?"
The Nanhu architecture has removed the loop predictor because the definition of FTB items in the current architecture will cause a conditional branch instruction to exist in multiple FTB items at the same time, which will make it difficult to accurately record the number of loops of a loop branch instruction. In the Yanqi Lake architecture, a prediction is made for each instruction, and a branch instruction will only appear once in the BTB, so there is no such problem.
<figure markdown>
![bpu](../figs/frontend/tage.png){ width="600px" }
<figcaption>TAGE basic logic diagram</figcaption>
</figure>
TAGE can mine extremely long branch history information by using multiple prediction tables with geometrically increasing history lengths. Its basic logic is shown in the figure above. It consists of a base prediction table and multiple history tables. The base prediction table is indexed by PC, while the history table is indexed by the XOR result of PC and a certain length of branch history folded. The branch history lengths used by different history tables are geometrically related. When predicting, the tag is calculated by XORing the PC and another folding result of the branch history corresponding to each history table, and matched with the tag read from the table. If the match is successful, the table hits. The final result depends on the result of the prediction table with the longest history length. In the Nanhu architecture, each prediction can predict up to 2 conditional branch instructions at the same time. When accessing each history table of TAGE, the starting address of the prediction block is used as PC, and two prediction results are taken out at the same time, and the branch history they use is also the same.
!!! note "Reason for using the same history prediction for two branches"
In theory, each branch should use the latest branch history, because in general, the bits closer to the current branch in the branch history sequence have a greater probability of affecting the result of the current branch. For the second branch, the latest branch history needs to include the result of the first branch. Here, the prediction of the two conditional branches can use the same branch history because if the jump result of the second branch is required, then the first branch must not jump, so the latest branch history bit must be 0 for the second branch, which is certain, so it does not introduce much information. The test results also show that the accuracy gain of using different branch history predictions for the two is almost negligible, but it introduces complex logic.

TAGE also has an [alternative prediction](#alt_pred) logic. We implemented the `USE_ALT_ON_NA` register with reference to the design of L-TAGE[^ltage] to dynamically decide whether to use the alternative prediction when the longest historical matching result is not confident enough. In the implementation, due to timing considerations, the result of the base prediction table is always used as the alternative prediction, which brings little accuracy loss.

The TAGE table entry contains a `useful` field. If its value is not 0, it means that the item is a useful item and will not be allocated as an empty item by the allocation algorithm during training. During training, we use a saturation counter to dynamically monitor the number of allocation successes/failures. When the number of allocation failures is large enough and the counter reaches saturation, we clear all `useful` fields.

SC is a statistical corrector. When it believes that TAGE has a high probability of misprediction, it will reverse the final prediction result. Its implementation basically refers to the structure of the O-GEHL predictor[^o_gehl], which is essentially a variant of the perceptron prediction logic[^perceptron].

!!! note "folded history"
Each history table of the TAGE class predictor has a specific history length. In order to index the history table after XOR with the PC, a long branch history sequence needs to be divided into many segments and then XORed together. The length of each segment is generally equal to the logarithm of the history table depth. Since the number of XORs is generally large, in order to avoid the delay of multi-level XOR on the prediction path, we will directly store the folded history. Since different length histories are folded in different ways, the number of folded histories required is equal to the number of (history length, folded length) tuples after deduplication. When updating a bit of history, you only need to XOR the oldest bit before folding and the latest bit to the corresponding position, and then do a shift operation.

### ITTAGE
In the RISC-V instruction set, the `jalr` instruction supports specifying the target address of unconditional jump instructions by adding an immediate value to the register value. Unlike the `jal` instruction that directly encodes the jump offset in the instruction, the jump address of `jalr` needs to be obtained indirectly through register access, so it is called an indirect jump instruction. Since the value of the register can be changed, the jump address of the same `jalr` instruction may be very different, so the mechanism of recording fixed addresses in FTB is difficult to accurately predict the target address of such instructions.
