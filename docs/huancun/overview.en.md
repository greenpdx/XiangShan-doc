# Overall architecture of L2/L3 Cache

L2/L3 Cache of XiangShan Nanhu architecture (i.e. [huancun subproject](https://github.com/OpenXiangShan/HuanCun)) is a directory-based non-inclusive cache (inclusive directory, non-inclusive data) designed with reference to [block-inclusive-cache-sifive](https://github.com/sifive/block-inclusivecache-sifive).

huancun uses Tilelink as the bus consistency protocol and can solve the [Cache Alias ​​problem](./cache_alias.md) when the L1 Cache is larger than 32KB by adding a custom Tilelink user-bit.

The overall structure of huancun is shown in the figure below:
![](../figs/huancun.png)

huancun can divide the index into slices according to the low bit of the requested address to improve concurrency. The number of [MSHR](./mshr.md) inside each Slice is configurable and is responsible for specific task management.

[DataBanks](./data.md) is responsible for storing specific data. The number of banks can be configured through parameters to improve read and write parallelism.

[RefillBuffer](./misc.md#refill_buffer) is responsible for temporarily storing Refill data so that it can be directly bypassed to the upper cache without writing through SRAM.

The Sink/Source\* related module is the Tilelink [channel control module](./channels.md), which is responsible for interacting with the standard Tilelink interface. On the one hand, it converts external requests into internal signals of the Cache, and on the other hand, it receives internal requests of the Cache and converts them into Tilelink requests and sends them to the interface.

In the [directory organization](./directory.md), huancun stores the directories of the upper data and the current data separately.
Self Directory/Client Directory are the directories corresponding to the current level Cache Data and the upper level Cache Data, respectively.

In addition, the [prefetcher](./prefetch.md) uses the BOP (Best-Offset Prefetching) algorithm, which can be configured or trimmed through parameters.

## The overall workflow of huancun is:

1. The [channel control module](./channels.md) accepts Tilelink requests and converts them into internal cache requests.

2. The [MSHR Alloc module](./misc.md#alloc) allocates an [MSHR](./mshr.md) for the internal request.

3. The [MSHR](./mshr.md) initiates different tasks according to the requirements of different requests. Task types include data reading and writing, sending new requests to upper and lower caches or returning responses, triggering or updating prefetchers, etc.

4. When all operations required for a request are completed in the [MSHR](./mshr.md), the [MSHR](./mshr.md) is released and waits for new requests.
