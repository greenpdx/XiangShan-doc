# Cache alias problem

In the NanHu architecture, both DCache and ICache use 128KB 8-way set associative structure, and the number of bits occupied by cache index and block offset has exceeded the page offset (the page offset of 4K page is 12 bits), which introduces the cache alias problem: when two virtual pages are mapped to the same physical page, the alias bits of the two virtual pages are likely to be different. If no additional processing is done, the two virtual pages will be located in different cache sets after VIPT indexing (Virtual Index Physical Tag), resulting in two physical blocks belonging to the same physical page being cached in the cache, causing some consistency errors.

![](../figs/huancun_cache_alias-1.jpg)

In order to allow L1 Cache to continue to use VIPT, XiangShan's cache system uses hardware to solve the cache alias problem. The specific solution is that L2 Cache ensures that a physical block has at most one alias bit in a VIPT cache at the upper layer.

The following is an example to illustrate how L2 solves the cache alias problem. As shown in the figure below, there is a block with a virtual address of 0x0000 in DCache. Virtual addresses 0x0000 and 0x1000 are mapped to the same physical address, and the aliases of these two addresses are different. At this time, DCache acquires the block with address 0x1000 from L2 and records the alias (0x1) in the user field of the acquire request. After reading the directory, L2 finds that the request is successful, but the alias (0x1) of the acquire is different from the alias (0x0) of the DCache at the physical address recorded by L2. Therefore, L2 will initiate a probe sub-request and record the alias (0x0) to be probed in the data field of the probe. After the probe sub-request is completed, L2 returns the block to DCache and changes the alias in the L2 client directory to (0x1).

![](../figs/huancun_cache_alias-2.jpg)
