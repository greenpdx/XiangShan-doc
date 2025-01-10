# Miss Queue

In the NanHu architecture, the Miss Queue contains 16 Miss Entries. Each Miss Entry is responsible for receiving miss load, store and atomic requests, retrieving the data to be backfilled from the L2 Cache, and returning the missing load data to the Load Queue.

Take load as an example, the processing flow of ordinary miss requests in the Miss Queue is as follows:

* Allocate an empty Miss Entry in the Miss Queue and record relevant information in the Miss Entry;

* Determine whether it needs to be replaced based on whether the block where `way_en` is located is valid. If it needs to be replaced, send a replace request to the Main Pipe;

* Send an Acquire request to L2 at the same time as sending the replace request. If it is an overwrite of the entire block, send AcquirePerm (L2 will save a sram read operation), otherwise send AcquireBlock;

* Wait for L2 to return permission (Grant) or data plus permission (GrantData);

* If it is a load miss, after receiving After each beat of GrantData, data should be forwarded to Load Queue;
* After receiving the first beat of Grant/GrantData, GrantAck is returned to L2;
* After receiving the last beat of Grant/GrantData and the replace request has been completed, a refill request is sent to Refill Pipe, and the response is waited for to complete the data backfill;
* Miss Entry is released.

The process of store miss is basically the same as that of load miss, except that the backfilled data does not need to be forwarded to Load Queue. In addition, after the backfill is finally completed, store miss returns a response to Store Buffer, indicating that the store is completed.

The processing flow of atomic instruction miss in miss queue is as follows:

* Allocate an empty Miss Entry in Miss Queue and record relevant information in Miss Entry;
* Send AcquireBlock request to L2;
* Wait for L2 to return GrantData;
* Return GrantAck to L2 after receiving the first beat of GrantData;
* Return GrantAck to L2 after receiving the last beat of GrantData Main Pipe sends a request, completes replacement and backfilling in Main Pipe at the same time, and returns a response to Miss Entry after completion;
* Release Miss Entry.

The above miss request processing flow is controlled by a set of status registers in Miss Entry, which will be introduced in [Miss Queue Status Maintenance](#miss-queue-status maintenance).

## Miss Queue Status Maintenance

Miss Entry is controlled by a series of status registers to complete which operations need to be completed and the execution order between these operations. As shown in the figure below, the `s_*` registers represent the requests to be scheduled, and the `w_*` registers represent the responses to be waited for. These registers are set to `true.B` in the initial state. When a Miss Entry is assigned to a request, the corresponding `s_*` and `w_*` registers are set to `false.B`. The former indicates that a certain outgoing request has not been sent out, The latter indicates that a response to be waited for has not been handshaked.

![dcache-miss-entry.png](../../figs/memblock/dcache-miss-entry.png)

<!-- <div align="center">
<img src=../../figs/memblock/dcache-miss-entry.png width=60%>
<div> -->

Each event in Miss Entry is executed in sequence according to certain dependencies. The figure above is a DAG flow chart. The arrow indicates that the next event can only be performed after the previous state register is set to `true.B`.

State|Description
-|-
`s_acquire`|Send AcquireBlock/AcquirePerm to L2. If the miss block needs to overwrite the entire block, only AcquirePerm is needed
`w_grantfirst`|Receive the first beat of GrantData
`w_grantlast`|Receive the last beat of GrantData beat
`s_grantack`|Indicates that after receiving the data from L2, a response is returned to L2. GrantAck can be returned when the first beat of Grant is received
`s_mainpipe_req`|Send the atomic request to Main Pipe and fill it back to DCache
`w_mainpipe_resp`|Indicates that after sending the atomic request to Main Pipe and filling it back to DCache, the response from Main Pipe is received
`s_replace_req`|Replacement is required. Before entering Miss Queue, load/store requests will select a replacement path according to the replacement algorithm. After entering Miss Queue, the request is directly sent to Replace Pipe
`w_replace_resp`|Complete replacement
`s_refill`|Load/store requests need to be sent to Refill Pipe for backfilling
`w_refill_resp`|Indicates that backfilling is completed.

## Miss Queue allocation logic

XiangShan's DCache Miss Queue supports a certain degree of request merging, thereby improving the efficiency of miss request processing. This section will introduce the allocation and merging strategy of Miss Entry in the NanHu architecture, and under what circumstances new miss requests should be rejected.

### Request merging conditions

When the block address of the allocated Miss Entry (request A) and the new miss request B is the same, they can be merged in the following two cases:

1. The Acquire request of request A has not yet been handshaked, and A is a load request, and B is a load or store request;

2. The Acquire of request A has been sent out, but the Grant (Data) has not been received, or the Grant (Data) has been received but has not been forwarded to the Load Queue, and A is a load or store request, and B is a load request.

The first condition can be merged because as long as the Acquire has not been handshaked, the various parameters of the Acquire request can be modified. It should be noted that a Miss Entry that handles load miss still needs to send the backfill data to the Load Queue after merging the store miss.

The second condition can be merged because, As long as the Miss Entry of the address has not sent the data to the Load Queue, a new load request can be merged in. After the Miss Queue gets the backfill data, it will wake up all the loads waiting for the data in the Load Queue at once.

### Request rejection conditions

New miss requests will be rejected by the Miss Queue in the following two cases:

1. The block address of the new miss request is the same as that of the block requested in a Miss Entry, but does not meet the [request merge condition](#request merge condition);

2. The block address of the new miss request is different from that of the block requested in a Miss Entry, but is located in the same slot in the DCache (that is, the two are located in the same set and way).

Case 1: Miss Entry can no longer merge miss requests with the same address for some reason, but the load pipeline and store pipeline (Main Pipe) cannot be blocked by the Miss Queue, so the Miss Queue needs to reject the miss request, and the load/store request will be resent after waiting for a period of time.

Case 2 Medium situation: In the NanHu version, in order to write the missing block from L2 to DCache immediately, the load/store miss request will determine the replacement way before entering the Miss Queue, so that after getting the backfill data, there is no need to do a tag comparison again and then decide the replacement way. This will introduce new problems: Assume that two load requests in the same set but with different tags miss in the Load Pipeline one after another, but the two loads decide to replace the same way and allocate a Miss Entry respectively, which eventually causes the block refilled later to overwrite the block refilled earlier. This will not only introduce performance problems, but also cause the dirty data to be lost if the block refilled earlier has dirty data, which will introduce correctness problems. Therefore, the second condition should be set to ensure that there are no two identical slot addresses in the Miss Queue.

### Miss Queue empty item allocation condition

When the new miss request meets the above [merge condition](#Request merge condition) or [rejection condition](#Request rejection condition), Just accept or reject the request accordingly. The conditions for merging and rejecting here are completely mutually exclusive, and there will be no conflict between the two.

Finally, if there is no Miss Entry that wants to merge or reject the new miss request:

* If the Miss Queue has an empty item, allocate a new Miss Entry;

* If the Miss Queue is full, reject the new miss request, and the request will be replayed after a certain period of time.

## Miss Queue triggers replacement

In the NanHu version, for load/store miss, the Miss Queue will determine the path to be replaced when the new Miss Entry is allocated, so that it can be backfilled immediately after receiving the block to be backfilled. To this end, DCache needs to replace in advance, at least read out the replacement block before backfilling occurs. Therefore, in this version, the Miss Entry can be replaced immediately after it is allocated, that is, a replace request is sent to the Main Pipe.

For performance considerations, we do not want the replacement block to be invalidated too early, so as to avoid the down access to L2/L3 During the time when the replacement block is accessed again in the core, resulting in a ping-pong effect and generating new unnecessary miss requests. Therefore, the replacement mentioned here does not really invalidate the replacement block, but reads out the data of the replacement block first and temporarily puts it in the write-back queue to sleep. During the sleep period of the replacement request, other requests can still access the replacement block in the DCache normally, as long as the write to the replacement block is synchronized to the write-back queue. When the backfill block is taken up, the sleeping block in the write-back queue can be awakened, and the write-back queue starts to release the replacement block downward. At the same time, the Miss Queue requests the Refill Pipe to complete the backfill, and the replacement block will be overwritten during the backfill.

## Miss Queue Refill

See [Refill Pipe](./refill_pipe.md).
