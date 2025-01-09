# Load Queue

This chapter introduces the design of the Nanhu architecture load queue of the Xiangshan processor.

The Nanhu architecture load queue is an 80-item circular queue. At most, it receives 4 instructions from dispatch

Receives the results of 2 instructions from the load pipeline to update its internal state

Receives the miss refill result from the dcache and updates the state of all instructions in the load queue waiting for this refill

Write back 2 miss load instructions. These instructions have already obtained the refill data and will compete with the normal memory access pipeline for two write-back ports

When the first-level data cache is refilled, all the data of an entire cacheline will be fed back to the load queue, and all instructions waiting for the data of this cacheline in the load queue will get the data.

## Contents of each item in the Load Queue

Each item in the Load Queue contains the following information:

* Physical address
* Virtual address (debug)
* Refill data item
* Status bit
* Trigger Status bits used

Status bits|Description
-|-
allocated|The item has been allocated by dispatch
datavalid|The load instruction has obtained the required data
writebacked|The result of the load instruction has been written back to the register stack and notified to ROB, RS
miss|The load instruction missed the dcache and is waiting for dcache refill
pending|The load instruction accesses the mmio address space and the execution is postponed. Waiting for the instruction to become the last instruction in the ROB
released|The cacheline accessed by the load instruction has been released by dcache (release)
error|The load instruction detected an error during execution

## Load Queue Enqueue

The load instruction enters the load queue in two steps: the early allocation of `enqPtr` and the actual writing of the load queue.

The reason for the early allocation is that the dispatch module is too far away from MemBlock, and sending the `enqPtr` generated in the load queue to dispatch as `lqIdx` requires a long delay. The Nanhu architecture is maintained near dispatch The pre-allocation logic of `enqPtr` is responsible for providing the `lqIdx` of the instruction.

<!-- ### Pre-allocation of enqPtr

!!! todo
See the dispatch section, [Pre-allocation](../../backend/dispatch.md) of `lqIdx` / `sqIdx`. -->

### Load Queue Actual Writing

When the load queue is actually written, the `enqPtr` of the load queue itself will be updated according to the number of instructions written to the load queue. For timing considerations, the load queue will only accept instructions dispatched by dispatch when the `number of empty items in the load queue >= the number of enq instructions`.

## Update Load Queue

During the execution of an instruction in the load pipeline, the state of the load queue can be updated at multiple stages.

### Load Stage 2

In this stage, dcache and forward recursion return results, and write the following content to the load In the queue:

* Key control signals
* Physical address
* Forward result
* Trigger check result

The status flags that will be updated at this time include: `datavalid`, `writebacked`, `miss`, `pending`, `released`

[MMIO](#mmio-uncached-access memory) The update method of the load queue is different from that of normal instructions.

### Load Stage 3

In this stage, some of the check return results that were not completed in the previous cycle are used by the load queue to update its status. For example, once the `dcacheRequireReplay` event (dcache requesting the instruction to be resent from the reservation station) is triggered, the `miss` and `datavalid` flags will be updated to false. This indicates that this instruction will be resent from the reservation station instead of waiting for refill in the load queue to wake it up. This operation occurs synchronously with the reservation station feedback operation in stage 3 of the load instruction pipeline.

!!! info
In the load pipeline [load The processing of [[miss](../fu/load_pipeline.md#load-miss) ... -->

![before-refill](../../figs/memblock/before-refill.png)

![after-refill](../../figs/memblock/after-refill.png)

After the load queue gets the data from the dcache refill, it can start writing back the missed load instructions from the load queue. The load queue provides two ports for writing back such instructions. In each cycle, the load queue selects the oldest instruction that has completed refill but has not been written back from the odd and even columns, and writes it back through the write-back port in the next cycle (for timing considerations, the selection of write-back instructions and the actual write-back are executed in the previous and next cycles). The load queue will compete with the load instructions that are normally executed in the load pipeline for the write-back port. When the instructions in the load pipeline try to write back, the write-back request from the load queue is blocked.

## Completion of load instructions

load The completion of an instruction refers to the operation of obtaining the result of load and writing it back to rob and rf. The selection of loads to write back to ROB and RF is divided into two parts: odd and even. The older instruction is selected first. At most two instructions are selected to be written back in each cycle. There are two types of instructions that the load queue selects to write back:

* Completed mmio load instructions
* Load instructions that previously missed and have now obtained the refilled data result from dcache

!!! info
Loads that normally hit dcache will be written back directly from the pipeline. See the [load pipeline](../fu/load_pipeline.md#stage-2) section.

The actual write-back operation occurs in the next beat of the write-back selection. The load queue will read the information of the corresponding instruction based on the selection result, complete the result clipping according to the instruction requirements, and finally compete for the load write-back port to write the result back. After the load is successfully written back, the `writebacked` flag will be updated to false.

!!! note
Pay attention to distinguish between load instructions written back to load queue and load are written back to rob and rf from the load queue. The two are different operations.

## Load submission related mechanism

After the instruction is submitted, rob generates an `lcommit` signal according to the number of load instructions submitted, notifying the load queue that these number of load instructions have been successfully submitted.

Since the load queue is far away from the ROB, the load queue actually uses `lcommit` to update the internal status two beats after the instruction commit is performed at the ROB. The load queue will update the `allocated` flag of the committed instruction to false to indicate its completion. At the same time, the queue tail pointer `deqPtr` is updated according to the number of submitted loads (`lcommit`).

## redirect

This section introduces the mechanism related to instruction redirection in the load queue. After the redirection reaches the load queue, the status of the load queue will be updated within 2 beats:

* Cycle1: Find all instructions on the wrong path according to robIdx. The allocated flag of the flushed instruction is set to false.
* Cycle2: According to the result of the previous search, count how many instructions need to be canceled and update enqPtr

Under the current design, after the jump instruction triggers the redirection, there may still be valid load instructions in the dispatch queue that need to enter the load queue. When updating enqPtr in cycle2, whether the instructions that enter the load queue during the redirection update of the load queue need to be canceled will be counted separately.

<!-- TODO: Processing of Cycle2 instruction enq -->

<!-- ### Queue pointer maintenance -->

## Store-load violation check related mechanism

While the store addr operation writes the address to the store queue (store addr pipeline stage 1), it also searches the load queue for load instructions with the same physical address but after the store. If these load instructions have been executed and produce incorrect results (i.e. triggering a store-load violation), the load queue will issue a redirection request: the processor will start from this load instruction and re-execute subsequent instructions from instruction fetch.

For two store pipelines, Each instruction needs to check whether the load instructions in three locations violate the rules (load pipeline stage 1 / stage 2, load queue). A total of 6 possible violations will be checked. The violation check logic needs to select the oldest one among the 6 possible violations and generate a redirection request. To this end, for each store instruction, its store-load violation check is divided into three cycles for execution. The operations performed in each cycle are as follows:

* Cycle 0: store addr updates the store queue
* The store pipeline sends the physical address to the load queue and generates a matching vector based on the address match
* Calculate the check range based on the lqIdx attached to the store instruction
* Check whether the instructions in the load pipeline stage 1 / stage 2 use the result of this store
* Cycle 1: Redirection generation
* Complete the violation check in the load queue
* If the previous instruction in the load pipeline stage 1 / stage 2 violated the rules, select the oldest one
* Cycle 2: Redirection generation
* Select the oldest one from all violations
* Generate redirect request out

```
stage 0: lq l1 l2 l1 l2 lq
| | | | | | (paddr match)
stage 1: lq l1 l2 l1 l2 lq
| | | | | | |
| |------------| |
| |
