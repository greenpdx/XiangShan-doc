# Store Queue

This chapter introduces the design of the store queue of the Nanhu architecture of the Xiangshan processor.

The store queue of the Nanhu architecture is a 64-item circular queue. At most, it can:

* Receive 4 instructions from dispatch
* Receive the address and control information of 2 instructions from the store addr pipeline
* ​​Receive the data of 2 instructions from the store data pipeline
* ​​Write the data of 2 instructions to the committed store buffer
* Provide the store to load forwarding result for 2 load pipelines

## Contents of each item in the Store Queue

Each item in the Store Queue contains the following information:

* Physical address
* Virtual address
* Data
* Data valid mask
* Status bit

Status bit|Description
-|-
allocated|The item has been allocated by dispatch
addrvalid|The store instruction has obtained the required address
datavalid|The store instruction has obtained the required data
committed|The store instruction has been committed
mmio|The store instruction accesses mmio Address space
pending|The store instruction accesses the mmio address space, and the execution is postponed. Waiting for the instruction to become the last instruction in the ROB

## Store Queue allocation and Enqueue

The store instruction enters the store queue in two steps: the early allocation of enqPtr and the actual writing of the store queue. The store processing flow here is similar to that of load, see the [load queue enqueue](./load_queue.md#load-queue-enqueue) section.

## Issue pointer issuePtr

`issuePtr` indicates that the data and address of the store instruction before this sqIdx are ready. The store queue provides this pointer to assist the reservation station in scheduling those load instructions that depend on the leading store instruction. issuePtr is an imprecise pointer (try to be as accurate as possible, but not mandatory).

## Store updates the Store Queue

### Store Addr

A store instruction will update the store in the store stage 1 / stage 2 of the sta pipeline queue. Among them, store stage 1 updates the address and most of the status bits. If there is no TLB miss or other problems, set the `addrvalid` flag. Stage 2 updates the `mmio` and `pending` flags. Among them, the reason why the `mmio` and `pending` flags are updated in stage 2 is that the timing of PMA checking whether the instruction is located in the MMIO area is tight.

### Store Data

The store data issued by the reservation station will be written to the store queue immediately. The `datavalid` flag will be set at the same time.

<!-- ### Special cases

* (1) For an mmio instruction with exceptions, we need to mark it as addrvalid (in this way it will trigger an exception when it reaches ROB's head) instead of pending to avoid sending them to lower level.
* (2) For an mmio instruction without exceptions, we mark it as pending. When the instruction reaches ROB's head, StoreQueue sends it to uncache channel. Upon receiving the response, StoreQueue writes back the instruction through arbiter with store units. It will later commit as normal. -->

## Store submission mechanism

After the instruction is submitted, rob generates `scommit` signal according to the number of store instructions submitted, notifying the store queue that these number of store instructions have been successfully submitted. Because the store queue is far away from the ROB. The store queue actually uses scommit to update the internal state two beats after the instruction is committed at the ROB.

The store queue will update the `committed` flag of the committed instruction to true to indicate that it has been submitted and can be written to the sbuffer.

## Store writes to Sbuffer

The submitted store instructions will not be cancelled, and will be read from the store queue and written to the sbuffer in sequence.

As the store queue continues to grow, the time to read data from the store queue becomes longer and longer. At the same time, the sbuffer needs to be checked in the process of receiving data written from the store queue. The logic of this check is very long (see [sbuffer](../lsq/committed_store_buffer.md#) for the description of sbuffer writing). For timing considerations, the Nanhu architecture will use a whole cycle to read data from the store queue. In the next cycle, the store queue will try to write the data read in the previous cycle into the sbuffer.

Before the store data is written to the sbuffer (which means that the sbuffer can provide the store to load forward result), this item in the store queue will remain valid to ensure that the store to load forward can correctly obtain the store result.

## Store to Load Forward

When the load instruction performs a store to load forward query, the sbuffer will provide the load instruction with the store data that was before this instruction but was not written to the sbuffer. Under the current [forward mechanism](../mechanism.md#store-to-load-forward), the forward from the store queue uses virtual address forward, Real address check.

### Generate forward results

<!-- Compare deqPtr (deqPtr) and forward.sqIdx, we have two cases:

* (1) if they have the same flag, we need to check range(tail, sqIdx)

* (2) if they have different flags, we need to check range(tail, LoadQueueSize) and range(0, sqIdx) -->

The process of generating **data** for forward delivery is as follows: Use real address query to generate real address match hit vector. Then, use the hit vector to generate forward results for each byte from the store queue. The forward data will be fed back to the load pipeline in the next beat generated by the forward request.

### Addr Valid Data Invalid

When the forward logic tries to pass the store data to the subsequent load, it may happen that the store address is ready but the data is not ready. The forward logic will notify the load pipeline when it detects this situation. stage1 feedbacks the result, and the [related logic in the load pipeline](../fu/load_pipeline.md#store-data-invalid) is responsible for subsequent processing.

### Forward pass correctness check

In order to ensure that the forward pass result generated by using the virtual address is the correct result, the store queue will perform virtual address forward pass related checks: once the matching results of the virtual and real addresses are different during forward pass, if the results are inconsistent, the inconsistent information will be fed back to the load pipeline. The load pipeline will perform [forward pass error processing](../fu/load_pipeline.md#forward-failure).

## Instruction redirection related mechanism

Similar to the load queue. See [Redirection logic in load queue](../lsq/load_queue.md#redirect).

## MMIO (uncached memory access)

The MMIO memory access mechanism of the store queue is similar to the [MMIO memory access mechanism of the load queue](../lsq/load_queue.md#redirect). [Access to Memory](../lsq/load_queue.md#mmio-uncached-%E8%AE%BF%E5%AD%98) is basically the same.

## Flush Store Queue

The store queue itself does not have flush logic, but the store queue will output empty and full signals to the outside. When the store queue needs to be flushed, the sbuffer will enter the refresh state and write all the data in it to the dcache until both the store queue and the sbuffer contain no valid data. During this process, the store queue will continuously write the data that has been submitted to the store command into the sbuffer.
