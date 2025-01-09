# Level 1 TLB

## Basic Introduction

TLB (Translation Lookaside Buffer) is responsible for address translation.
Xiangshan supports the Sv39 mode specified in the manual, and does not support other modes for the time being. The length of the virtual address is 39 bits, and the length of the physical address is 36 bits, which can be parameterized.
Whether the virtual memory is enabled is determined by the current privilege level (such as M mode or S Mode, etc.) and the Satp register, etc. This judgment is completed inside the TLB and is transparent to the outside of the TLB. Therefore, from the perspective of modules outside the TLB, all addresses have undergone address translation by the TLB.

## Organizational Form

Before accessing memory in the core, both the front-end instruction fetch and the back-end memory access need to be translated by the TLB. Due to the long physical distance and to avoid mutual contamination, the front-end ITLB (instruction TLB) and the back-end memory access DTLB (Data TLB) are two TLBs. ITLB adopts full associative mode, 32 items of normal pages (NormalPage) are responsible for 4KB pages, and 8 items of super pages (SuperPage) are responsible for 2MB and 1GB address conversion. DTLB adopts mixed mode, 64 items of normal pages are directly associative and responsible for 4KB pages, 16 items of super pages are fully associative and responsible for pages of all sizes, and adopts pseudo LRU replacement strategy. The normal page part provides capacity, but has low flexibility and is prone to conflict; the super page part has strong flexibility but small capacity. Therefore, the normal page part is used as the Victim of the super page part. When backfilling, the super page part is backfilled first, and the items evicted from the super page part are written to the normal page part (only for 4KB pages).

Memory access has 2 Load pipelines and 2 Store pipelines. For timing considerations, they need a separate TLB. At the same time, for performance considerations, the contents of these TLBs need to be consistent, similar to mutual prefetching. Therefore, these 4 DTLBs use an "external" replacement algorithm module to ensure content consistency.

## Blocking and non-blocking

The front-end instruction fetch requires blocking access to ITLB, that is, when TLB misses, TLB does not return the miss result immediately, but performs Page Table Walk to retrieve the page table entry and then returns.
The back-end memory access requires non-blocking access to DTLB, that is, when TLB misses, TLB also needs to return the result immediately, whether it is a miss or a hit.
In the Yanqihu architecture and Nanhu architecture, the TLB body is non-blocking access, and the blocking access of the front-end instruction fetch is completed by the front-end module. The replay, TLB does not store the request information.

New version architecture preview: TLB can parameterize the blocking form of each request port, statically select blocking or non-blocking, and one module can adapt to the needs of the front and back ends at the same time.

## Sfence.vma and ASID

When the Sfence.vma instruction is executed, it will first clear all the contents of the Store Buffer (write back to the DCache), and then send a refresh signal to each part of the MMU. The refresh signal is unidirectional and will only last for one beat, with no return signal. The instruction will finally refresh the entire pipeline and re-execute from instruction fetch.
Sfence.vma will cancel all inflight requests, including Repeater and Filter, as well as inflight requests in L2 TLB, and refresh the cached page tables in TLB and L2TLB according to the address and ASID.
The fully associative part is refreshed based on whether it hits, and the set associative (direct associative) part is refreshed directly based on the index.

The Yanqihu architecture does not support ASID (Address Space IDentifier), and the Nanhu architecture adds support for ASID, with a length of 16, which can be parameterized.
All inflight requests do not carry ASID information, which is due to the following reasons:

1. Generally, the ASID of the inflight request is the same as the ASID field of the Satp

2. When switching ASID, the inflight request is "speculative" access, which is almost "useless"

Combining these two reasons, when switching ASID, all inflights will be cancelled, and inflight does not need to carry ASID information.

## Update A/D bits

According to the manual, the A (Access) and D (Dirty) bits of the page table will need to be updated. Xiangshan uses the method of reporting page fault exceptions and updating by software.
