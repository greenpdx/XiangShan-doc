# Store Pipeline

This chapter introduces the design of the store addr / store data pipeline of the Xiangshan processor (Nanhu architecture) and the processing flow of the store instruction.

The Xiangshan processor (Nanhu architecture) adopts the execution mode of separating the store address and data. The store address and data can be sent out from the reservation station separately when ready and enter the subsequent processing flow. The two data/address operations corresponding to the same store instruction are linked together by the same robIdx / sqIdx.

The Xiangshan processor (Nanhu architecture) contains two store addr (sta) pipelines, each store addr pipeline is divided into 4 pipeline stages. The first two pipeline stages are responsible for passing the control information and address of the store to the store queue, and the last two pipeline stages are responsible for waiting for the memory access dependency check to be completed.

The Xiangshan processor (Nanhu architecture) contains two store data (std) pipelines. After the reservation station provides store data, the store data pipeline will immediately write the data into the corresponding item of the store queue.

## Sta Pipeline

The division of each level of pipeline is as follows:

![storepipe](../../figs/memblock/store-pipeline.png)

Stage 0

* Calculate virtual address
* Send virtual address to TLB

Stage 1

* TLB generates physical address
* Complete fast exception check
* Start memory access dependency check
* Send physical address to store queue

Stage 2

* Memory access dependency check
* Complete all exception checks and PMA queries, and update store queue according to the results

Stage 3

* Complete memory access dependency check
* Notify ROB that instructions can be submitted

[Store Queue](../lsq/store_queue.md) describes the details of sta updating store queue.

## Std Pipeline

Stage 0

* The reservation station gives store data
* Write store data to store queue

## Store

**TLB miss handling.** Like the load pipeline, the store addr pipeline may also experience TLB miss. The handling methods of the two are basically the same, see [load TLB miss handling]( ./load_pipeline.md#tlb-miss). Store addr only uses one rsFeedback port to feedback to the reservation station whether the store addr calculation operation needs to be resent from the reservation station.

**PMA and exception check.** For timing considerations, the MMIO and other check results are not completed until the data is updated to the store queue for one cycle. At this time, the store pipeline will write the final result of the query into the store queue.
