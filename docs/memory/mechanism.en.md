# Out-of-order memory access mechanism

This chapter introduces the key mechanism for implementing out-of-order memory access in the Xiangshan processor.

## Load Hit

When a load instruction hits, the load instruction will go through 3 stages after being issued from the reservation station:

* Stage 0: Calculate address, read TLB, read dcache tag

* Stage 1: Read dcache data

* Stage 2: Get read result, select and write back

After stage 2, there will be an additional stage to handle the status update that stage 2 did not have time to complete. For details on the operations performed at each stage, see the detailed introduction of [load pipeline](./fu/load_pipeline.md).

## Sta and Std

The address calculation part of the store instruction will go through 4 stages after being issued from the reservation station. For details, see [sta pipeline](./fu/store_pipeline.md#Sta-Pipeline):

* Stage 0: Calculate address, Read TLB
* Stage 1: addr and other control information are written to the store queue, and violation checking begins
* Stage 2: Violation checking
* Stage 3: Violation checking, allowing store instructions to be submitted

After the data calculation part of the store instruction is emitted from the reservation station, the data will be directly moved from the reservation station to the store queue, see [std pipeline](./fu/store_pipeline.md#Std-Pipeline).

For details on the operations performed at each stage, see the detailed introduction of [load pipeline](./fu/load_pipeline.md).

## Load Miss processing

See [Load Miss](./fu/load_pipeline.md#Load-Miss).

## Replay From RS

This section introduces the mechanism of replaying load instructions and sta operations from the reservation station.

Take the load instruction as an example. When some events occur, we will replay these loads from the reservation station. Instructions:

* TLB miss
* L1 DCache MSHR full
* DCache bank conflict
* Address match found during forward pass but data not ready (Data invalid)

The common characteristics of these events are:

* Low frequency of occurrence (compared to normal memory access instructions)
* Memory access instructions cannot be executed normally when these events occur
* These events will not occur when the same memory access instruction is executed again after a period of time
* For example, TLB miss events will disappear after PTW completes TLB refill

The purpose of the reissue mechanism from the reservation station is to let these instructions wait for a while in the reservation stack and re-execute after a certain cycle. This mechanism is implemented as follows: After an instruction is issued from the memory access RS, it still needs to be retained in the RS. When the memory access instruction leaves the pipeline, it will feedback to the RS whether it needs to be reissued from the reservation station. The instructions that need to be reissued from the reservation station will continue to wait in the RS and re-issue after a certain time interval.

Currently, there are two ports in the load pipeline that feedback to the reservation station whether the instruction needs to be reissued. These two ports are located in the load stage 1 (`feedbackFast`) and load stage 3 (`feedbackSlow`). Instructions that can be detected in load stage 0 and load stage 1 will feedback the reissue request to the reservation station through the `feedbackFast` port of load stage 1. Reissue requests that can only be detected in load stage 2 will be fed back to the reservation station through the `feedbackSlow` port of load stage 3. The design of the two ports is to allow the reservation station to reissue some instructions that need to be reissued earlier.

After the `feedbackFast` port generates a reissue request, the corresponding instruction will not continue to flow in the pipeline. That is, the `feedbackSlow` port will not generate feedback for this instruction.

The store addr (sta) pipeline only sets one feedback port. At store stage 1, the store pipeline will report to the reservation station whether the instruction needs to be reissued.

In addition to the information on whether to reissue the instruction, the reissue feedback port also includes the following information:

* Use reservation station index (rsIdx) indexes the position of the instruction to be reissued in the reservation station
* Use the sourceType field to distinguish different reissue reasons
* Provides an interface for feedback of this store sqIdx when the load finds that the previous store address is ready but the data is not ready

!!! note
This mechanism may change in the next version of the design.

## Store To Load Forward

Store to Load Forward (STLF) refers to the operation that the subsequent load instruction that accesses the same address obtains the data of this store instruction from the memory access queue and buffer in the core before the data of the store instruction is written to the data cache.

The forward operation of store to load is assigned to the three-stage pipeline execution. During the forward operation, the forward logic will check in parallel whether there is data required by the current load in the committed store buffer and store queue. If so, the data will be merged into the result of this load.

### Virtual address forward

For timing considerations, the Nanhu architecture uses a virtual address forward and real address check mechanism. This mechanism is achieved by placing the TLB The query is removed from the data path of data forwarding (but still retained in the control path) to optimize the timing performance of store to load forward.

<!-- !!! todo -->
<!-- Basic idea, diagram -->

**Data path for virtual address forwarding:** Stage 0 of the [load pipeline](../memory/fu/load_pipeline.md) generates the mask used for data forwarding according to the sqIdx of the instruction. In stage 1 of the load pipeline, the virtual address and mask are sent to the [store queue](../memory/lsq/store_queue.md#store-to-load-forward-query) and [committed store buffer](../memory/lsq/committed_store_buffer.md#store-to-load-forward-query) for forward query. In stage 2 of the load pipeline, the store queue and committed store buffer Generate forward query results, which will be merged with the results read from dcache.

**Control path for virtual address forward pass:** In stage 0 of the load pipeline, the virtual address of the instruction is sent to the TLB to start the virtual-to-real address conversion. In stage 1 of the load pipeline, the TLB feeds back the physical address of the instruction. The physical address and mask are sent to the [store queue](../memory/lsq/store_queue.md#store-to-load-forward-query) and [committed store buffer](../memory/lsq/committed_store_buffer.md#store-to-load-forward-query) for forward query (address matching only). In load stage 2, the matching result of the virtual address and the matching result of the real address will be compared. Once the two are different, it means that an error has occurred in the virtual address forward pass. [After checking for errors, trigger rollback and refresh the committed store buffer](../fu/load_pipeline.md#forward-failure). Such an operation will exclude the virtual address that caused the error from the store queue and committed store buffer.

!!! info
In contrast, the process of real address forward delivery in Yanqi Lake architecture is as follows: stage 0 of the load pipeline will generate the mask used for data forward delivery according to the sqIdx of the instruction. In stage 1 of the load pipeline, the TLB feeds back the physical address, and this physical address and mask are sent to the store queue and committed store buffer for forward query. In stage 2 of the load pipeline, the store queue and committed store buffer generate forward query results, which are merged with the results read from the dcache. Both the control and data paths follow this process.

### Preservation of forward results

If DCache misses, keep the forward results. The forward results (mask and data) will be written to the load queue. When the dcache refills the results later, the load queue will be responsible for merging the refills The data from the previous instruction and the result of the forward instruction are combined to generate a complete load result.

### Performance optimization related to forward delivery

When the dcache misses but the forward delivery is completely hit, the result of this instruction can be written back directly without waiting for the dcache to return the data. However, due to timing considerations (there is no time to mark this instruction as a hit state), the Nanhu architecture leaves this situation to the load queue. Such instructions will directly set the `datavalid` flag (indicating that the load data is valid) when updating the load queue. As a result, the load queue will immediately find that such instructions do not need to wait for the result of the dcache refill. Such instructions can be directly selected and written back.

## Store Load Violation

This section introduces the check and recovery of store-load violations. The load violation check begins when the store instruction reaches stage 1. If a load violation is found during the check, the store that triggered the load violation will not be marked as *committable* in the ROB. At the same time, the rollback operation will be triggered immediately without waiting for the triggering load Violated store instruction submission. The load queue section introduces the [detailed process of checking and redirecting](../memory/lsq/load_queue.md#store---load-%E8%BF%9D%E4%BE%8B%E6%A3%80%E6%9F%A5%E7%9B%B8%E5%85%B3%E6%9C%BA%E5%88%B6).

## Load Load Violation

See [load-load violation check and recovery](../memory/lsq/load_queue.md#load---load-%E8%BF%9D%E4%BE%8B%E6%A3%80%E6%9F%A5%E7%9B%B8%E5%85%B3%E6%9C%BA%E5%88%B6)

## load Contention of write-back ports

The Southlake architecture provides two load write-back ports. This port is responsible for writing the result of the load back to the reservation station, register stack, and notifying the ROB that the instruction has been executed. Both stage 2 of the load pipeline and the load queue can use this port to write back the result. Both will compete for the right to use this port.

Under normal circumstances, the load instruction in the pipeline has a higher priority.
