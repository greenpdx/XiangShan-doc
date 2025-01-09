# MMU

This section introduces Xiangshan's MMU (Memory Management Unit), including TLB, L2TLB, Repeater, PMP and PMA.

In order to achieve process isolation, each process will have its own address space, and the addresses used are all virtual addresses. MMU, or Memory Management Unit, is mainly responsible for translating virtual addresses into physical addresses in the processor, and then using this physical address to access memory. At the same time, permission checks are also performed, such as whether it is writable or executable.

Xiangshan processor supports Sv39 paging mechanism, the main feature of which is that the virtual address length is 39 bits, the lower 12 bits are the page offset, and the middle 27 bits are divided into three segments, that is, three-level page tables, which means that the page table is traversed three times for memory access, so the page table needs to be cached.

In Xiangshan's MMU, the ITLB and DTLB, which are located at the front and back ends respectively and are tightly coupled with the pipeline, need to consider the needs of the pipeline and timing issues, such as splitting into two beats. If ITLB and DTLB miss, they will send a request to L2 TLB. If L2 TLB also misses, Hardware Page Table Walker will be used to access the page table content in memory. L2 TLB mainly considers how to improve parallelism and filter repeated requests. Repeater is the request buffer from the first level TLB to L2 TLB. PMP and PMA need to perform permission checks on all physical address accesses.

![mmu-overall](../../figs/memblock/mmu-overall.png)

## TLB

Xiangshan's TLB can configure the organizational structure, including associative mode, number of items, and replacement strategy. The default configuration is ITLB 32 items of normal pages and 4 items of large pages (SuperPage), fully associative, pseudo LRU replacement; DTLB is 64 items of normal pages directly connected, 16 items of fully associative responsible for pages of all sizes, pseudo LRU replacement strategy. The directly associated items of DTLB are victim caches of fully associated items.

DTLB is a non-blocking design. If it misses, the request needs to be resent by the outside (Load/Store Unit). ITLB is a layer wrapped on the basis of TLB, realizing a simple blocking design.
Each memory access pipeline has an independent DTLB, but the contents of Load TLB and Store TLB are the same. Multiple TLBs are maintained to keep the same content through a unified "replacement algorithm" and backfill information:

## L2TLB

L2 TLB is a larger page table cache shared by ITLB and DTLB, divided into five parts: Page Cache, Pake Walker, Last Level Page Walker, Miss Queue and Prefetcher.

The request of the first level TLB will first be sent to the Page Cache. If it hits, it will be returned to the first level TLB. If it misses, it will enter the Page Walker or Last Level Page Walker for query according to the query situation of the Page Cache. If the Page Walker or Last Level Page Walker is already occupied, it enters the Miss Queue to wait for resources.
At the same time, in order to speed up page table access, Page Cache caches all three levels of page tables. Page Cache supports ecc verification. If the ecc verification fails, it refreshes this item and re-performs Page Walk.

Page Walker receives requests from Page Cache and performs Hardware Page Table Walk. Page Walker only accesses the first two levels of page tables, i.e. 1GB and 2MB, and does not access the 4KB page table. The access to the 4KB page table is undertaken by the Last Level Page Walker. If the Page Walker accesses a leaf node, i.e. a large page or an empty page, it returns to the first level TLB, otherwise it is sent to the Last Level Page Walker. The Page Walker can only process one request at a time.

The Last Level Page Walker receives requests from Page Cache and Page Walker to access the last level of page table. The Last Level Page Walker can process N requests at the same time, where N is the number of items in the Last Level Page Walker. Because the access to the page table is concentrated on the last level page table, the L2 TLB mainly improves the parallelism of memory access through the Last Level Page Walker, but at the same time limits the memory access capability of the Page Walker to avoid repeated memory access.

The Miss Queue receives requests from the Page Cache and the Last Level Page Walker, and acts as a buffer for miss requests, waiting to re-access the Page Cache.

The result of the Page Cache triggers the generation of prefetch requests. The processing of prefetch requests is similar to that of ordinary first-level TLB requests. The difference is that the prefetch request will not be returned to the first-level TLB; when the prefetch request enters the Miss Queue, if the first two levels of page tables miss, it will be directly discarded to prevent excessive use of Page Cache and Page Walker resources. The NextLine prefetch algorithm is currently used.

## Repeater

Because the TLB and L2 TLB have a relatively long physical distance, it will cause a relatively long line delay, so it is necessary to add a beat in the middle, which is called Repeater.
Filter is enhanced on the basis of Repeater, accepts the request of DTLB, and sends it to L2 TLB. Filter is responsible for filtering out duplicate requests to avoid duplicate items in the first-level TLB. The number of items in the filter determines the parallelism of the L2 TLB to a certain extent.

## PMP and PMA

Xiangshan implements PMP (Physical Memory Protection), which is divided into four parts: CSR Unit, ITLB, DTLB, and L2TLB. In the CSR Unit, it is responsible for reading and writing. In the ITLB, DTLB and L2 TLB, it is responsible for writing and address checking. In the DTLB, due to timing factors, it is divided into two parts: dynamic checking and static checking. When backfilling the DTLB, a PMP check is performed and the result is stored in the DTLB. The result of the PMP is queried together with the TLB check, which is a static check. After the TLB is translated, the physical address is obtained, and then a PMP check is performed, which is a dynamic check. In order to implement static checking, the PMP granularity is increased to 4KB.

PMA is a custom implementation that uses a PMP-like method and uses the reserved bits of the PMP Configure register to implement cacheable and Atmoic checks. PMA and PMP are queried in parallel. In addition, a Memory Mapped PMA is provided for use by peripherals such as DMA.
