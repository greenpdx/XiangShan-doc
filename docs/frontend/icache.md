# Instruction Cache Documentation
<!-- This diagram needs to be redrawn -->
![icache](../figs/frontend/ICache.png)

This chapter describes the implementation of the Xiangshan processor instruction cache.

## Instruction cache configuration
| Parameter name | Parameter description |
| ----------- | ------------------------------------ |
| `nSet` | Number of instruction cache Sets, default is 256 |
| `nWays` | Number of group-connected ways for each instruction cache Set, default is 8 ways |
| `nTLBEntries` | Number of ITLB entries, default is 40 entries (32 normal pages + 8 large pages) |
| `tagECC` | Meta SRAM check method, Nanhu version is configured as parity check |
| `dataECC` | Data SRAM check method, Nanhu version is configured as parity check |
| `replacer` | Replacement strategy, default is random replacement, Nanhu version is configured as PLRU |
| `hasPrefetch` | Instruction prefetch switch, default is off |
| `nPrefetchEntries` | Number of instruction prefetch entries, and the maximum number of cache line prefetches that can be supported at the same time, default is 4 |

## Control Logic
<!-- Internal Logic Diagram of the Main Control Logic Module MainPipe of the Instruction Cache: -->

The Main Control Logic Module MainPipe of the Instruction Cache consists of a 3-stage pipeline:

- In the `s0` stage, an instruction fetch request containing two cache lines is sent from the FTQ, which contains a signal whether each request is valid (instruction packets that do not cross lines will only send a read request for one cache line). At the same time, MainPipe will extract the request address as a cache set index and send it to the [storage part](#imem) of the instruction cache. On the other hand, these requests will be sent to the ITLB for [instruction address translation](#itlb)
- In the `s1` stage, the storage SRAM returns a cache set with a total of N cache ways (cache line metadata and data). At the same time, ITLB returns the physical address corresponding to the request. Next, the main control logic intercepts the physical address and matches it with the cache tags of N ways, generating two results: cache hit and cache miss. In addition, the cache line to be replaced is selected based on the status information of the replacement algorithm.
- In the `s2` stage, the hit request directly returns the data to the IFU. When a miss occurs, the pipeline needs to be paused and the request is sent to the miss processing unit MissUnit. After the MissUnit is filled and returns the data, the data is returned to the IFU. In this stage, the physical address translated by the ITLB is also sent to the PMP module for access permission query. If the permission is wrong, an instruction access exception (Instruction Access Fault) will be triggered.

<h2 id=itlb> Instruction Address Translation </h2>

Since the instruction cache uses the VIPT (Virtual Index Physical Tag) cache method, the virtual address needs to be translated into a physical address before the address tag comparison. In the `s0` stage of the control pipeline, the virtual addresses of the two cacheline requests are sent to the query port of the ITLB at the same time, and the ITLB returns a signal whether the virtual address hits in this clock cycle. If it hits, the corresponding physical address will be returned in the next beat. If it does not hit, the control logic will block the MainPipe pipeline and wait until the ITLB refills and returns the physical address.

## Cache miss processing

The request that misses will be handed over to the MissUnit to send a Tilelink `Aquire` request to the downstream L2 Cache. After the MissUnit receives the `Grant` request for the corresponding data, if the cache line needs to be replaced, the MissUnit will send a Release request to the ReplacePipe. The ReplacePipe will re-read the SRAM to get the data, and then send it to the ReleaseUnit to initiate a `Release` request to the L2 Cache. Finally, the MissUnit refills the SRAM, waits until the refill is completed, and returns the data to the MainPipe, and the MainPipe returns the data to the IFU.

A cache line miss may occur in any of the two requests, so two missEntry items are set in MissUnit to improve concurrency.

## Exception handling
There are two main exceptions generated in ICache: Instruction Page Fault reported by ITLB and Access Fault reported by ITLB and PMP. MainPipe will report the exception information directly to IFU, and the requested data is considered invalid.

<h2 id=imem> Storage section </h2>

The storage logic of the instruction cache is mainly divided into Meta SRAM (storing the tag and consistency status of each cache line) and Data SRAM (storing the content of each cache line). Parity check code is supported internally for data verification. When a check error occurs, a bus error will be reported and an interrupt will be generated. Meta/Data SRAM is divided into parity banks internally. Two adjacent cache lines in the virtual address space will be divided into different banks to implement the reading of two cache lines at a time.

## Coherence Support

The instruction cache of Xiangshan Nanhu architecture implements the coherence protocol defined by Tilelink. It mainly handles Tilelink `Probe` and `Release` requests by adding an additional pipeline ReplacePipe.

The ReplacePipe of the instruction cache consists of 4 pipelines:

- In the `r0` stage, it receives the `Probe` request sent from the ProbeUnit and the `Release` request sent from the MissUnit, and also initiates a read of the Meta/Data SRAM. Because the request here contains the virtual address and the actual physical address, there is no need to do address translation.
- In the `r1` stage, ReplacePipe, like MainPipe, uses the physical address to match the address of the N-way cache line of a Set returned by SRAM, generating two signals, hit and miss. This signal is only valid for `Probe`, because the `Release` request must be in the instruction cache.
- In the `r2` phase, the hit `Probe` request will modify the permissions of the corresponding cache block, and will send this request to ReleaseUnit to send a `ProbrResponse` request to L2. The permissions of this cache line are generated based on the original permissions (T/B) and the permissions of the Probe (toN, toB, toT). The miss request will not make permission changes, and will be sent to ReleaseUnit to report to L2 that the permissions have been changed to NToN (there is no data required by `Probe` in the instruction cache). The `Release` request will also be sent to ReleaseUnit to send `ReleaseData` to L2. And only `Release` requests are allowed to enter `r3`
- In the `r3` phase, ReplacePipe reports to MissUnit that the replaced block has been `Release`ed downward, notifying MissUnit that it can be refilled.

## Instruction prefetching

The Xiangshan Nanhu architecture implements a simple `Fetch Directed Prefetching (FDP)`[^fdp], which allows branch prediction to guide instruction prefetching. For this purpose, an instruction prefetcher `IPrefetch` is added. The specific prefetch mechanism is as follows:

* A prefetch pointer is added to the instruction fetch target queue. The pointer is located between the prediction pointer and the instruction fetch pointer. The prefetch pointer reads the target address of the current instruction packet (if it jumps, it is the jump target, if it does not jump, it is the start address of the next packet in sequence) and sends it to the prefetcher.

* The fetcher will complete the address translation and access the Meta SRAM of the instruction cache. If it finds that the address is already in the instruction cache, the prefetch request will be canceled. If not, it will apply for an allocation to `PrefetchEntry`, send a Tilelink `Hint` request to the `L2` cache, and prefetch the corresponding cache line to L2.
* To ensure that prefetch requests are not sent repeatedly to L2, the prefetcher records the physical addresses of prefetch requests that have been sent. Any request will check this record before applying for `PrefetchEntry`. If it is found that it is the same as the prefetch request that has been sent, the current prefetch request will be canceled.

## Quote
[^fdp]: Reinman G, Calder B, Austin T. Fetch directed instruction prefetching[C]//MICRO-32. Proceedings of the 32nd Annual ACM/IEEE International Symposium on Microarchitecture. IEEE, 1999: 16-27.

--8<-- "docs/frontend/abbreviations.md"
