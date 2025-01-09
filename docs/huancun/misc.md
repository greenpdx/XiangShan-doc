# Request Buffer Design

In order to avoid resource competition and mutual interference when multiple concurrent requests are processed in the Cache at the same time, huancun blocks requests with Set as the granularity.
Requests with the same priority (three priorities, A, B, and C) and the same Set cannot enter the MSHR at the same time. In order to reduce the performance loss of the Cache caused by this blocking strategy, we designed the Request Buffer to buffer blocked requests, so that subsequent requests from different Sets can enter the MSHR without blocking.

Considering the characteristics of the number of Tilelink bus requests in the CPU, A>>B, A>>C, the Request Buffer only buffers requests from the A channel, which can
improve performance while reducing the complexity of hardware implementation.

The design of the Request Buffer is similar to the reservation station/transmission queue in the CPU. When a new request cannot enter the MSHR, it will try to enter the Request Buffer. If there is an empty item in the Buffer, the request will be assigned to the corresponding position, and it will also record which MSHR the request is waiting for to be released (`wait_table` in the code). When MSHR is released, its MSHR id will be broadcasted to the Request Buffer, and the items in the Buffer that are dependent on the MSHR can be "woken up".

When there are multiple requests of the same Set in the Request Buffer, in order to ensure that these requests can leave the Buffer in FIFO order, the Buffer also maintains a **dependency matrix** `buffer_dep_mask`, which records the dependencies between the items in the Buffer.

`buffer_dep_mask[i][j]` is 1, which means that the request at position `i` and the request at position `j` are both from the same Set, and the request at position `i` arrives later than the request at position `j`, so when the external MSHR is released, request `j` should leave the Buffer first.

# Refill Buffer Design <a name="refill_buffer"></a>

In order to reduce the delay of Cache Miss, huancun uses Refill Buffer to buffer the data refilled from the lower cache or memory,
so that the refilled data can be directly returned to the upper cache without writing to SRAM first.

# MSHR Alloc (MSHR allocation module) <a name="alloc"></a>

According to the Tilelink manual specification, in order to avoid deadlock in the system, high-priority requests must be able to interrupt low-priority requests to execute first,
therefore huancun designed N (N >= 1) abc MSHRs, 1 b MSHR, and 1 c MSHR.

In the absence of Set conflict, all requests select an empty item in abc MSHR to enter;
When a newly arrived high-priority request conflicts with an existing low-priority request in MSHR, the high-priority request will enter the dedicated MSHR.
For example: if the b request conflicts with the a request Set in the abc MSHR, the new b request will be assigned to the b MSHR.

If the c request conflicts with the a/b request in the abc/b MSHR, the new c request will be assigned to the c MSHR.

# ProbeHelper

Since huancun adopts the design of inclusive-directory non-inclusive data, when the Client Directory cannot store a new Cache block (e.g. blockA) due to limited capacity, the cache at this level needs to send a Probe request to the upper level, probe a Cache Block (e.g. blockB) of the upper level, and then store the corresponding state of the new Cache Block (blockA) in the Client Directory.

In order to simplify the processing flow of a single MSHR, we designed ProbeHelper to monitor the reading results of the Client Directory. If a capacity conflict is found, ProbeHelper generates a fake B request to probe the target block from the upper level.

In this way, there is no need to consider the capacity conflict of Client Directoy in a single MSHR, which simplifies the design.
