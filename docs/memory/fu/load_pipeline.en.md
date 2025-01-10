# Load Pipeline

This chapter introduces the design of the load pipeline of the Nanhu architecture of the Xiangshan processor and the processing flow of the load instruction. The Xiangshan processor (Yanqi Lake architecture) contains two load pipelines, each of which is divided into three pipeline stages.

<!-- !!! todo
update graph -->
<!-- ![loadpipe](../../figs/memblock/load-pipeline.png) -->

## Pipeline division

The load instruction execution pipeline is divided into the following stages:

### Stage 0

* Instructions and operands are read from the reservation station
* The adder adds the operand to the immediate value and calculates the virtual address
* The virtual address is sent to the TLB for TLB query
* The virtual address is sent to the data cache for Tag indexing

### Stage 1

* TLB generates physical address
* Complete fast exception check
* Virtual address performs Data indexing
* The physical address is sent to the data cache for tag comparison
* The virtual/physical address is sent to the store queue / committed store buffer to start the forward pass operation from store to load
* According to the hit vector returned by the first-level data cache and the result of the preliminary exception judgment, an early wake-up signal is generated and sent to the reservation station
* If an event that causes the instruction to be reissued from the reservation station occurs at this level, notify the reservation station that this instruction needs to be reissued (`feedbackFast`)

### Stage 2

* Complete the exception check
* Select data based on the first-level data cache and the result returned by the forward pass
* Perform a trimming operation on the returned result according to the requirements of the load instruction
* Update the status of the corresponding item in the load queue
* The result (integer) is written back to the common data bus
* The result (floating point) is sent to the floating point module

### Stage 3

* According to the feedback result of the dcache, update the status of the corresponding item in the load queue
* According to the feedback result of the dcache, feedback to the reservation station and notify the reservation station whether this instruction needs to be reissued (`feedbackSlow`)

!!! info
Stage 3 is responsible for sending out some stage 2 check results that are delayed due to timing reasons.

## Load Miss

A load instruction that misses will perform the following operations to obtain the data it needs. This section will introduce these mechanisms one by one:

* Disable early wakeup of the current instruction
* Update the status of the load queue
* Allocate dcache MSHR (MissQueue entry)
* Listen to dcache refill
* Write back the missed load instruction

In load stage 1, based on the dcache tag comparison result, the load pipeline can know whether the current instruction misses. When a miss occurs, the early wakeup valid bit of this instruction will be set to `false` to **disable early wakeup of the current instruction**.

In load stage 2, if a miss is found, the load pipeline will not write the result back to the register stack and will not occupy the load instruction write-back port. At the same time, **update the status of the load queue**, and this load instruction that has missed will be in the The load queue waits for dcache refill. Meanwhile, dcache will try to allocate dcache MSHR (MissQueue entry) for this miss load instruction. Due to the complex allocation logic, the allocation result can only be fed back to the load queue in the next beat.

In load stage 3, **the status of the load queue is updated again according to the result of dcache MSHR allocation**. If dcache MSHR allocation fails, the reservation station is requested to resend this instruction.

If this instruction is successfully allocated dcache MSHR, it will subsequently listen to the result of dcache refill in the load queue. See [load queue: Load Refill](../lsq/load_queue.md#load-refill).

## Replay From RS

The load pipeline is **non-blocking**, that is, no matter what abnormal situation occurs, it will not affect the flow of valid instructions in the pipeline. When an abnormal situation other than load miss occurs and causes the load to fail to complete normally, the load The pipeline will replay this instruction using the [Replay From RS mechanism](../mechanism.md#Replay-From-RS). The following are the events that trigger the replay from reservation station mechanism:

### TLB miss

**TLB miss** event will request replay from reservation station by using feedbackSlow port. TLB miss instruction will have replay delay when replaying, and it will be replayed after waiting in reservation station until the delay is over. The replay delay exists because TLB refill takes time, and replaying instruction before TLB refill is completed will cause TLB miss, which is meaningless.

### Bank Conflict

**bank conflict** event. Bank conflict event includes bank conflict between two load pipelines, and conflict between load pipeline and store operation writing cacheline (store operation here refers to the committed store from committed store buffer to dcache). Currently, we only allow two load instructions to be replayed without triggering bank

### Dcache MSHR Allocate Failure

**dcache MSHR allocation failure**. For the reasons of dcache MSHR allocation failure, see [dcache/MissQueue](../dcache/miss_queue.md). For retransmission caused by dcache MSHR allocation failure, the retransmission request is sent using the feedbackSlow port. There is no delay for retransmission from the reservation station. The reservation station can immediately retransmit this instruction when receiving the dcache MSHR allocation failure retransmission request.

!!! note
The design here needs to be optimized. The dcache MSHR allocation failure may come from several different reasons. Separately handling these situations will help improve performance.

### Store Data Invalid

**Store address is ready but data is not ready**. These instructions will not update the load queue or write back, but will issue a reissue request through feedbackSlow in load stage 3 to notify the reservation station that this instruction is **waiting for the data of a previous store instruction to be ready**. During the store-load forward check, the sqIdx of the store that the load depends on will be checked and fed back to the reservation station through the feedbackSlow port. The reissue generated in this way has a non-fixed delay. The reservation station can wait until the corresponding store data is generated according to the found sqIdx before reissuing this load instruction.

## Exception handling

The exceptions that the load pipeline can handle are divided into two categories: exceptions from address checking and exceptions from error handling. Exceptions use a separate exception path, and the timing is looser than the data path. The address check results are generated in stages to optimize the timing performance, see [MMU](../mmu/mmu.md) section.

## Prefetch instruction processing

Currently, software prefetch instructions use a similar processing flow to load instructions. Software prefetch instructions will enter the load pipeline like normal load instructions. When a miss is found, a request is sent to the MissQueue of dcache to trigger access to the lower cache. In particular, all exceptions will be blocked during the execution of software prefetch instructions, and no reissue will be made.

## Early wakeup of load

Nanhu's reservation station supports an early wakeup mechanism to schedule subsequent instructions as soon as possible. However, Nanhu architecture currently **does not support the speculative wakeup mechanism**. The instructions that are woken up in advance must be able to execute normally, otherwise the entire pipeline needs to be flushed. If a load instruction is executed normally but no early wakeup signal is issued, the subsequent instructions that depend on this load will be issued one cycle later, resulting in a slight performance loss.

The load pipeline will give a fast wakeup signal to the reservation station at load stage 1. Due to The line delay between MemBlock and IntBlock, this signal is on the critical path. When the early wake-up signal is generated, the load pipeline will make a rough judgment on whether the instruction can be executed normally. Once the instruction shows signs of not being able to be executed normally, it will not be woken up in advance.

In extreme cases, the load instruction will mistakenly issue an early wake-up, and the entire pipeline needs to be flushed. The condition for the load instruction to mistakenly issue an early wake-up is: the load instruction has an exception after load stage 1 (due to data error/bus error).

## Forward Failure

This section introduces the processing of the South Lake architecture when the virtual address forward delivery fails. When the store queue or store buffer feedback virtual address forward delivery fails, the load pipeline will attach the label `replayInst` (need to be reissued from instruction fetch) to this instruction in stage 2. Redirection is triggered when this instruction reaches the end of the ROB queue. Since virtual address forward delivery failure is a very rare phenomenon, such redirection will not have a significant impact on performance.

## Debug related

The trigger mechanism is set in the load pipeline. For timing considerations, the Southlake architecture only supports using addresses as trigger conditions.
