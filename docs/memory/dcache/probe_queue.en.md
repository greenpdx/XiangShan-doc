# Probe Queue

The Probe Queue contains 16 Probe Entries, which are responsible for receiving Probe requests from L2 Cache, converting Probe requests into internal signals and sending them to Main Pipe, and modifying the permissions of the Probe block in Main Pipe.

## Scheduling of Probe Requests

The processing flow of the Probe Queue after receiving the Probe request from L2 is as follows:

* Allocate an empty Probe Entry;

* Send a probe request to Main Pipe, which will be delayed by one beat due to timing considerations;

* Wait for Main Pipe to return a response;

* Release the Probe Entry.

## Supplement

### Alias ​​processing

The NanHu architecture uses a 128KB VIPT cache, which introduces the cache alias problem. The NanHu architecture solves the alias problem from a hardware perspective. As shown in the figure below, the L2 Cache directory maintains the alias bits corresponding to each physical block. (i.e. the part where the virtual address index exceeds the page offset), ensuring that a certain physical address has only one alias bit in the L1 Cache. When the L1 Cache wants to obtain different alias bits, i.e. different virtual indexes, on a certain physical address, the L2 Cache will probe the cache block corresponding to another virtual index.

![dcache-probe-queue-alias.jpg](../../figs/memblock/dcache-probe-queue-alias.jpg)

Since the Probe requests sent by L2 are all based on the physical address, but L1 will access the cache in VIPT mode, L1 also needs to know the alias bit of the probed block. The NanHu architecture uses the data field of the B channel in the TileLink protocol to transmit the 2-bit alias bit. After receiving the request, the Probe Queue will combine the alias bit and the page offset to obtain the index needed to access the DCache.

### Atomic instruction support

In In XiangShan's design, atomic operations (including lr-sc) are completed in DCache. When executing the LR instruction, it is ensured that the target address is already in DCache. At this time, in order to simplify the design, LR will register a reservation set in the Main Pipe, record the address of LR, and block the Probe to the address. In order to avoid deadlock, Main Pipe will not block Probe after waiting for SC for a certain period of time (determined by parameters `LRSCCycles` and `LRSCBackOff`). At this time, any SC instruction received is regarded as SC fail. Therefore, after LR registers the reservation set and waits for SC pairing, it is necessary to block Probe requests to operate on DCache.
