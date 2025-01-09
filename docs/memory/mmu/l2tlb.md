# Second Level TLB

L2 TLB is a larger page table cache, but it also includes a Page Table Walker unit, which is responsible for Hardware Page Table Walk.

## Structural Design

L2 TLB contains four main units:

1. Page Cache: caches page tables. All three levels of Sv39 page tables are cached separately, and the query of three levels of information can be completed in one shot

2. Page Walker: queries the first two levels of page tables in memory, which is essentially a state machine

3. Last Level Page Walker: queries the last level of page table in memory

4. Miss Queue: caches the miss requests of Page Cache and Last Level Page Walker

5. Prefetcher: prefetcher

Requests from the first level TLB will first access the Page Cache. If it hits, it will be directly returned to the first level TLB. If it is missing, it will enter the Page Walker, Last Level Page Walker or Miss Queue.
The page table can be traversed through Page Walker and Last Level Page Walker; the resource can be waited for through Miss Queue so as to access Page Cache again. After accessing the page table, it can be directly returned to the first-level TLB.
The prefetcher monitors the query results of Page Cache and generates prefetch requests. The prefetch results will be stored in Page Cache and will not be returned to the first-level TLB.

## Page Cache

Page Cache is a "true" L2 TLB with a larger cache capacity. It caches 3 layers of page tables and caches them separately, so 3 layers of page tables can be accessed in one shot. Determine whether it hits based on the virtual address and get the result closest to the leaf node.
Page Cache only stores valid page table entries. Because the memory access width is 512 bits, that is, 8 page tables, one Page Cache entry contains 8 page tables (1 virtual page number, 8 physical page numbers, and 8 permission bits).

Page Cache access is divided into four steps. The first step is to send a read signal, the second step is to get the read result and cache it for one step, the third step is to determine whether it is a hit and perform ECC check, and the fourth step is to return the result.
If the ECC check fails, no exception or interrupt signal will be sent to the core, but the current item will be invalidated, and the miss result will be returned, and the Page Walk will be performed again.
Page Cache returns the information it can provide, including:

1. Whether the leaf node (including 4KB page table and large page) is hit and the hit page table, returned to the first level TLB

2. Whether the hit item is prefetched by the prefetcher, returned to the prefetcher for use

3. Whether the first two levels of page tables are hit and the hit result, returned to the Page Walker or Last Level Page Walker for use

## Page Walker

Page Walker is a state machine that accesses the page table according to the virtual address. The access rules are similar to those in the RV manual until:

1. Accessing the 2MB node, returned to the Last Level Page Walker, and the Last Level Page Walker accesses the last level of the page table

2. Accessing the leaf node, which is a large page (SuperPage), returned to the first level TLB

3. Accessing an illegal node, the v bit of the page table entry is false, etc.

The results of the Page Walker memory access will be stored in the Page Cache.
Page Walker can only process one request at a time and can only access the first two levels of page tables at most. Its memory access capability is weak. The reason is explained in the Last Level Page Walker section.

## Last Level Page Walker

Last Level Page Walker exists to improve the memory access capability of L2 TLB. Last Level Page Walker does not merge repeated requests, but records them and shares the memory access results to avoid repeated memory access.
Last Level Page Walker receives miss requests from Page Cache and the results of Page Walk to access the last level of page table.
Requests from Page Cache can access memory if they only lack the last level of page table. Otherwise, they need to access Page Cache again until the address of the last level of page table is obtained.
Requests from Page Walker can access memory because they have obtained the first two levels of page tables and only lack the last level of page table.

Last Level Page Walker and Page Walker work together to complete the entire process of Page Table Walk. In order to improve the parallelism of memory access, the Last Level Page Walker configures different IDs for requests and has multiple inflight requests at the same time. However, the first two levels (1GB, 2MB) of different requests may be the same, and considering that the miss probability of the first two levels is lower than that of the last level page table, it is not considered to improve the access parallelism of the first two levels of page tables, and only one Page Walk is set to reduce the complexity of the design.

## Miss Queue

The essence of Miss Queue is a buffer queue that plays the role of waiting for resources. Miss Queue can receive requests from Page Cache or Last Level Page Walker and wait to access Page Cache again.

## Prefetch

Currently, the NextLine prefetch algorithm is used. When there is a miss or hit but the hit item is a prefetch item, the next prefetch request is generated.
Prefetch requests access Page Cache. Because Page Walker has weak memory access capabilities, if a prefetch request misses, the prefetch request will not enter Page Walker or Miss Queue, but will be directly discarded. If the prefetch request is only missing the last section of the page table, the Last Level Page Walker can be accessed.
The prefetch result is stored in the Page Cache and is not returned to the first-level TLB.
