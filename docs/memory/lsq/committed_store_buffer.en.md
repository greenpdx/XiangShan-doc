# Committed Store Buffer

This chapter introduces the design of the committed store buffer (sbuffer) of the Nanhu architecture of the Xiangshan processor.

The Committed Store Buffer of the Nanhu architecture can store up to 16 items of data organized in cacheline units. Each cycle of the Committed Store Buffer can at most:

* Receive 2 store instructions written by the store queue
* Write a cacheline to the data cache

The Committed Store Buffer merges store write requests with cacheline as the granularity. When the capacity exceeds the set threshold, the Committed Store Buffer will perform a swap operation, use PLRU to select the cacheline to be written to the data cache, and write it to the data cache. The automatic swap threshold of the Committed Store Buffer can be set using the `smblockctl` CSR.

<!-- TODO: Figure -->

## Contents of each item in sbuffer

* Status bit
* Virtual address
* Data
* Data valid mask

Status bit|Description
-|-
state_valid|This item is valid
state_inflight|A write request has been issued to dcache, and the write result is being waited for by dcache
w_timeout|wait for timeout, wait for the resend timer to timeout, to reissue a write request to dcache
w_sameblock_inflight|with sameblock inflight, there is an sbuffer item of the same cacheline being written to dcache. Instructions in this state will not be selected to be written to dcache to maintain the order of sbuffer write operations to dcache. See [Selection of items to write to dcache](#Selection of items to write to-dcache-)

## Sq write mechanism to sbuffer

Due to timing, it takes two beats for sq to write data to sbuffer.

* First beat: Read data and address from sq and temporarily store them in `DatamoduleResultBuffer`, see [store queue](../lsq/store_queue.md).
* Second shot: Use the address temporarily stored in the previous shot to compare with the address/status already in sbuffer to determine whether it can be written. If it can, actually perform the write operation.

The address and status check performed in the second shot will produce the following results:

Check result|Subsequent operation
-|-
Address does not match, this cacheline does not exist in sbuffer|Allocate a new item and write
Address matches, this item has not initiated a write dcache request|Merge the data written this time into the existing item in sbuffer
Address matches, this item is writing dcache request|Block write request
sbuffer has no empty item|Block write request

If the write or merge is successfully executed, the data and mask of the corresponding item will be updated. The address and control information will also be written to sbuffer during the first write. For timing considerations, when there is no empty item in sbuffer, even if the data of the same cacheline can be merged, Writing also happens.

## sbuffer write mechanism to dcache

sbuffer writes to dcache in two steps:

* First step: select the item to be written to dcache, read the data, and the corresponding item is marked as being written to dcache

* Second step: write request is sent to dcache

After sbuffer sends a write request to dcache, it will wait for the write result returned by dcache. dcache may not accept the write request (miss and MSHR is full, see dcache mainpipe section), at which time the store instruction will start the retransmission timer in sbuffer and wait. When the retransmission timer returns to 0, sbuffer will try to write this item to dcache again. Only after dcache reports that the write of an item is successfully hit, sbuffer will mark this item as invalid.

There are two types of feedback interfaces from dcache to sbuffer: Hit interface and resend interface. The hit interface may come from MainPipe (hit) or RefillPipe (missed, but written after being assigned to MissQueue entry). The resend interface also comes from MainPipe (missed and not assigned to MissQueue entry). For details, see the description of [dcache mainpipe](../dcache/main_pipe.md).

During the writing of sbuffer to dcache, the state change process of an item in sbuffer is as follows:

1. Valid, not written to dcache
1. Valid, writing to dcache
1. Failed to write to dcache, waiting for resend or invalid

!!! note
The design of sbuffer allows the data of the same cacheline to be written to sbuffer while an item in sbuffer is being written to dcache. The new data and the data being written to sbuffer data will occupy sbuffer

### Selection of items to write to dcache

The sbuffer items that meet the following conditions will be written to dcache by sbuffer, and the conditions are arranged in descending order of priority:

1. The dcache request retransmission reaches the waiting time

1. sbuffer refresh

1. The time an item exists in sbuffer reaches the threshold and times out

1. sbuffer is in a state close to full, and some items are selected to be swapped out

The selection of swapped out items uses the PLRU algorithm. When the store writes to the sbuffer, or when the store merges with an item in the sbuffer, the PLRU will be updated. When the `w_sameblock_inflight` flag of an item is high (which means that another sbuffer entry used by the same cacheline is being written to dcache), this item will not be selected as the object to be written to dcache.

### Flush Sbuffer

On active flushing sbuffer, sbuffer will write all committed instructions in the store queue to dcache. sbuffer will also refresh itself to support the virtual address forward mechanism. When sbuffer finds that the result of virtual and real address forwarding conflicts (see [forward support](./committed_store_buffer.md#store-to-load-forward-query)), sbuffer will also automatically refresh.
<!-- The number of cycles that trigger automatic swapping does not support manual configuration for the time being. -->

## Store to Load Forward Query

When the load instruction performs a store to load forward query, sbuffer will also provide the load instruction with the data of the store that has been committed but not written to dcache. Under the current [forward mechanism](../mechanism.md#store-to-load-forward), the forward mechanism from sbuffer uses virtual address forwarding and real address checking.

### Generate forward results

sbuffer The data generated by the virtual address forward pass will be fed back in the next cycle when the forward pass query request arrives, which is consistent with the store queue.

### Forward pass correctness check

In order to ensure that the forward pass result generated by using the virtual address is the correct result, the store buffer will perform virtual address forward pass related checks:

<!-- * **Check during forward pass.** -->

Once the matching results of the virtual and real addresses are different during forward pass, the sbuffer is triggered to force refresh. All items are written to dcache to prevent sbuffer from continuing to generate incorrect forward pass results. At the same time, the message of forward pass failure is fed back to the load pipeline. The load pipeline performs [forward pass error processing](../fu/load_pipeline.md#forward-failure).

<!-- * **Check during write.** The write operation of the store buffer will try to merge the write operation of the same cacheline into one item of the sbuffer according to the real address. If it is found that the real address is the same but the virtual address is different, the sbuffer is triggered to force refresh. The corresponding item in sbuffer will be updated to the new virtual address. -->

<!-- The write-time check mechanism can be considered to be cancelled, and the sbuffer will be refreshed lazily only when a problem is found in the previous pass. -->
