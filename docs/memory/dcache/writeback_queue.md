# Writeback Queue

The DCache of the NanHu architecture uses an 18-entry writeback queue, which is responsible for releasing replacement blocks (Release) to the L2 Cache through the C channel of TL-C, or responding to Probe requests (ProbeAck), and supports merging Release and ProbeAck to reduce the number of requests and optimize timing.

## Writeback Queue Entry

For timing considerations, when the wbq is full, new requests will be rejected regardless of whether they can be merged; when the wbq is not full, all requests will be accepted. At this time, either an empty entry is allocated to the new request, or the new request is merged into the existing Writeback Entry. Later in the [Status Maintenance] (#writeback-queue-Status Maintenance) section, you will see that the Writeback Entry can be merged into a new Release or ProbeAck request at any time. Therefore, in the NanHu architecture, the only thing to determine whether the writeback queue can be enqueued is whether there are empty items in the queue, which greatly shortens the logical delay of enqueuing.

## Writeback Queue status maintenance

Each Writeback Entry has the following 4 states:

Status|Description
-|-
`s_invalid`|This Writeback Entry is empty
`s_sleep`|Prepare to send Release request, but temporarily sleep and wait for refill request to wake up
`s_release_req`|Sending Release or ProbeAck request
`s_release_resp`|Waiting for ReleaseAck request

!!! info
**Description of `s_sleep` status**: For performance considerations, we do not want the replacement block to be invalidated too early, so as to avoid the core accessing the replacement block again during the time of accessing L2/L3 downward, resulting in ping-pong effect and generating new unnecessary miss requests. Therefore, the replacement request enters the Main Pipe and Writeback Queue successively, not to really invalidate the replacement block, but to read the data of the replacement block first and temporarily put it in the writeback queue to sleep. During the sleep period of the replacement request, other requests can still access the DCache normally To replace the block in the write-back queue, just synchronize the write of the replacement block to the write-back queue. When the backfill block is taken up, the sleep block in the write-back queue can be awakened, and the write-back queue starts to release the replacement block downward. At the same time, the Miss Queue requests the Refill Pipe to complete the backfill. The replacement block will be overwritten during the backfill.

If the request is not merged, the process of Writeback Entry processing ProbeAck and processing Release is as follows:

1. ProbeAck: `s_invalid` -> `s_release_req` -> `s_invalid`

2. Release: `s_invalid` -> `s_sleep` -> `s_release_req` -> `s_release_resp` -> `s_invalid`

If there are requests that can be merged, the processing flow will be slightly more complicated:

3. Release merge ProbeAck: It is possible to receive ProbeAcks with the same address at any stage in the above Release processing flow: If it is In the `s_sleep` phase, the Release request is directly converted to a ProbeAck request; if it is in the `s_release_req` phase, and the Release request has not completed the handshake, the Release can also be directly converted to a ProbeAck; if it is in the `s_release_req` phase, but the Release request has completed at least one handshake, it means that the ProbeAck request is too late, and the `release_later` bit will be set at this time, and the ProbeAck related information will be recorded. After all Releases are processed, ProbeAck will be processed.

4. ProbeAck merges Release: Since this part is very detailed, it will not be introduced in more detail. For specific content, please refer to the code of `src/main/scala/xiangshan/cache/dcache/mainpipe/WritebackQueue.scala`. The main idea is to merge if possible, and try to do as few Releases as possible. If Release comes too late, set the `release_later` bit and process it later.

## Block Miss Queue Request

The TileLink manual's restrictions on concurrent transactions require that if the master has a pending Grant (i.e., it has not yet sent a GrantAck), it cannot send a Release with the same address. Therefore, when all miss requests enter the Miss Queue, if they are found to have the same address as an item in the Writeback Queue, the miss request will be blocked.
