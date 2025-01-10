# Level 1 Data Cache

DCache is tightly coupled with the memory access pipeline in the Nanhu architecture, and interacts directly with the L2 Cache through the TileLink bus protocol. The default DCache in the Nanhu architecture is 128KB total capacity, 8-way set associative structure, pseudo LRU replacement and SECDED check. DCache works with L2 cache to handle the [cache alias problem](../../huancun/cache_alias.md) caused by the 128KB capacity. The dcache of the Nanhu architecture supports custom [cache operations](./csr_cache_op.md).

DCache internal modules include:

Module name|Description
-|-
[Load PipeLine](./load_pipeline.md) (Load pipeline)|tightly coupled with the Load memory access pipeline, 3-beat readout
[Main Pipeline](./main_pipe.md) (Main pipeline)|Responsible for the execution of store, probe, replace, atomic operations
[Refill Pipeline](./refill_pipe.md) (Refill pipeline)|Responsible for writing L2 refill data back to dcache
[Atomics Unit](../fu/atom.md#dcache-Support for atomic instructions) (AtomicsReplayEntry)|Schedule atomic requests
[Miss Queue](./miss_queue.md) (MSHRs, 16 items)|Request missing blocks from L2, each item controls the flow of the miss request in DCache through a state machine
[Probe Queue](./probe_queue.md) (8 items)|Receive consistency requests from L2 Cache
[Writeback Queue](./writeback_queue.md) (18 items)|Responsible for writing replacement blocks back to L2 Cache, or respond to Probe requests

[Committed Store Buffer](../lsq/committed_store_buffer.md) will also serve as a retransmission buffer for write requests while sending write requests to DCache. The overall structure diagram of DCache and Committed Store Buffer is as follows:

![dcache](../../figs/memblock/dcache.png)

## Interface

DCache in the Yanqi Lake architecture uses the TileLink bus protocol for L2 Cache. The consistency requests involved in TileLink are mainly divided into three categories:

* Acquire Acquisition authority
* Probe Passive release authority
* Release Active release authority

<!-- !!! todo
Figure to be updated -->

<!-- ![tilelink](../../figs/memblock/dcache-tilelink-interface.jpg) -->

## Request processing flow

### Load request processing flow

* After three levels of pipeline, if it hits, it returns directly, and if it fails, it enters the Miss Queue
* The Miss Queue retrieves the backfill data and forwards it to the Load Queue
* In the main pipeline, the backfill data is written to the DCache
* If a block needs to be replaced, the replacement block is written back in the Writeback Queue

### Store request processing flow

The Store Replay Unit receives the request from the Store Buffer and accesses the DCache in the main pipeline:

* If the DCache hits
* Complete the write to the DCache in the Main Pipeline
* ​​If the DCache does not hit and the Miss Queue is full (or refuses to accept the request)
* The Store Buffer resends this request after a period of time
* If the DCache does not hit and the Miss Queue successfully accepts the request
* The Miss Queue continues to perform subsequent operations
* After completion, notify the Store Buffer and update the DCache through the Refill Pipe
* If there is a replaced block, write it back in the Writeback Queue
* In particular, the replaced block is written down by the Writeback Queue only after the block that replaces it reaches the DCache.

### Atomics request processing flow

See [Processing flow of atomic instructions](../fu/atom.md).

<!-- ### Refill request processing flow

!!! todo
To be updated -->

<!-- ### Replace trigger and processing flow -->

### Probe request processing flow

* Receive Probe request from L2 Cache

* Modify the permissions of the probed block in the main pipeline

* Return a response and write back the dirty data at the same time

<!-- ## Resource contention

!!! todo
Components in dcache will compete for resources such as data array, tag array, MissQueue, Writeback Queue, etc. This section will describe how dcache schedules these resources. -->
