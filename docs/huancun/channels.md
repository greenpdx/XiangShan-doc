# Tilelink Channel Control

This chapter will introduce the structure of each channel control module in huancun. Please be familiar with the TileLink bus protocol before reading.

The channel control module is divided into Sink module and Source module, among which,

* The Sink module receives active requests and passive responses on the TileLink bus. For active requests (SinkA, SinkB, SinkC), the request is converted into a huancun internal request and sent to the MSHR Alloc module or Request Buffer. For passive responses, the response is processed and fed back to the corresponding MSHR (SinkD, SinkE)

* The Source module receives the internal request of MSHR, and sends it to the TileLink bus after processing and packaging (SourceA, SourceB, SourceC, SourceD, SourceE)

In addition, some modules will also receive additional Tasks sent from MSHR to complete some related tasks.

## SinkA

SinkA forwards the request received from channel A to MSHR Alloc. If there is no data, it is forwarded directly; if there is data (such as PutData, etc.), it needs to be stored in PutBuffer. Each Beat of the data is stored in the free item in PutBuffer in the form of an array, and the free item index is forwarded to MSHR Alloc.

The data stored in PutBuffer will be used by SourceD or SourceA. When the Put request hits in the cache, SourceD is responsible for writing it to DataStorage; when it misses, SourceA is responsible for forwarding it directly to the next level of cache or memory.

When receiving a new request, channel A will be blocked because MSHR Alloc is not ready or PutBuffer is full (when there is data); once a request with data is received, it will not block the reception of its subsequent Beats.

## SinkB

SinkB forwards the request received from channel B to MSHR Alloc, and there is no other logic.

## SinkC

SinkC receives requests from the upper layer to release permissions or data, including Release/ReleaseData released actively by the Client and ProbeAck/ProbeAckData responded to passive release, and forwards the request to MSHR Alloc.

The processing of ReleaseData/ProbeAckData is similar to the processing flow of SinkA for PutData request. SinkC maintains a Buffer, and each Beat of the data is stored in the free item in the Buffer in the form of an array, and forwards the index to MSHR Alloc. SinkC receives Tasks from MSHR to process the data in the Buffer. Tasks are divided into three types, namely

* Save: Store the data items in the Buffer into DataStorage
* Through: Release the data items in the Buffer directly to the lower-level Cache or memory after packaging
* Drop: Discard the data items in the Buffer

When receiving a Task, SinkC enters the Busy state and does not receive subsequent tasks until the Task is processed. If the Task has not received the data Beats that need to be processed, it will be blocked (controlled by beatValsThrough/beatValsSave).

## SinkD

SinkD receives the Grant/GrantData and ReleaseAck responses from the D channel, uses the Source field to query MSHR to get the set and way, and returns the Resp signal to MSHR when receiving the first or last Beat. For responses with data, the data from the Grant will be sent to DataStorage and [RefillBuffer](misc.md) at the same time.

## SinkE

Receives GrantAck from the E channel and sends Resp to MSHR. There is no other logic.

## SourceA

Receives Acquire and Put requests from MSHR and forwards them from the A channel. For Put requests, it reads data from the PutBuffer in SourceA and then forwards it downward.

## SourceB

SourceB receives Tasks from MSHR and sends Probe requests through the B channel. SourceB maintains a status register workVec internally. Each bit of workVec corresponds to a client that supports Probe. When workVec is empty, it can receive requests from MSHR and mark all clients that need Probe in the workVec register. After marking, it sends Probe to these clients in turn and clears the mark bits in turn.

## SourceC

SourceC receives Tasks from MSHR and is responsible for sending read requests to DataStorage. After receiving data, it stores it in the queue and sends it out from the C channel after dequeuing.

The reason for setting a queue is that SourceC's pipeline does not have a blocking mechanism, and DataStorage does not have blocking and cancellation functions. This requires that once a Task is allowed to be received, it must be able to be processed and enter the Queue. Back pressure control is used here to ensure that even if the C channel is always blocked, the space in the Queue can accommodate all items that may enter: including the impending SRAMLatency items in the pipeline plus the beatSize items obtained by the current Task to read dataStorage.

## SourceD

SourceD is the most complex of the channel control modules. It has two main responsibilities: reading data and sending it back to the upper level through the D channel, and sending write requests to DataStorage to process Put and other requests.

Level 1 sends a read request to DataStorage or RefillBuffer based on the Task information, and sets Busy to block new Tasks. New Tasks are allowed to enter only after the request of the last Beat is sent.

Level 2 first sends a read request to PutBuffer of SinkA, and stores the data in the queue after receiving it; if the Task does not need the data or the data is bypassed from RefillBuffer, this level sends a response directly through the D channel.

Level 3 is connected to Level 2 through a Pipe, that is, it enters Level 3 only after DataStorage returns data. For general requests, this level sends a response to the read data through the D channel; for Put requests, this level sends an AccessAck response when processing the first Beat.

Level 4 sends a write request to DataStorage

#### Bypass check

For the same request, SourceD must have the lowest priority. The priority of each channel in DataStorage is actually determined according to the task priority of the same request. However, since MSHR will be released before the request to SourceD is completed, the priority rule in DataStorage will be violated. For Acquire requests, it is released when GrantAck is received, but the GrantAck of the Client can be given when Grant first is received, so before Grant last, MSHR will release it when it receives GrantAck. For Get requests, GrantAck is not required. As long as SourceD sends it, MSHR will release it. In these early release situations, Get and Acquire may still be reading data from SourceD when the next request for the same Set comes and issues a write operation to this Set. From the perspective of correctness, the previous read must be done first, but SourceD has the lowest priority in DataStorage. Therefore, the solution is to pull out SourceD's Set/Way and compare it with the channel to be written. If the Set/Way is the same, block them and let SourceD do it first.

## SourceE

Receive the request from MSHR and send GrantAck from the E channel.
