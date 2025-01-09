# L2 prefetcher design

XiangShan's L2 Cache uses Best-Offset hardware prefetching. BOP is an offset-based prefetcher. The offset can be adjusted dynamically as the program is executed. Its main purpose is to ensure the timeliness of prefetching.

For the specific prefetching algorithm, please refer to the paper. The following only introduces the part related to XiangShan's specific implementation.

<!--
You may need a picture here
-->
First, a prefetch bit is used in the L2 directory to record whether a cache block is a prefetched block. When MSHR receives an Acquire request from DCache, if the requested address does not hit in L2 or hits the prefetched block, MSHR will initiate a "trigger prefetch" request, which will be sent to the prefetcher. The prefetcher will add the best offset trained by the Best-offset algorithm to the request address to generate the prefetch address, and then the prefetcher will send the prefetch request to the bank where the address is located. The request type is Intent. The MSHR allocator allocates an MSHR for the prefetch request. If the prefetch block is not in L2, MSHR will be responsible for fetching the prefetch block from L3.

When MSHR completes a prefetch, it will send another response to the prefetcher, and the prefetcher will train the Best-offset algorithm based on this response.

<!--
TODO: Remember to quote! ! !
-->
