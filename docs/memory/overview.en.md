# Overall architecture

The `Memblock` of the Xiangshan processor contains the memory access pipeline and queue in the core, as well as the first-level data cache tightly coupled with the memory access pipeline. We use the term `memory access unit` to refer to this part of the `Memblock`.

![Overall pipeline](../figs/memblock/nanhu-memblock.png)

The memory access unit in the core of the Xiangshan processor (Nanhu architecture) is shown in the figure above. It contains two [load pipelines](./fu/load_pipeline.md), two separate [sta pipelines](./fu/store_pipeline.md#Sta-Pipeline) and two [std pipelines](./fu/store_pipeline.md#Std-Pipeline). [load queue](./lsq/load_queue.md) and [store queue](./lsq/store_queue.md) Responsible for maintaining the order information of memory access instructions. The load queue is responsible for monitoring the subsequent refill results and performing write-back operations when the load instruction is missing in the first-level cache. The store queue is responsible for temporarily storing the store data before the instruction is committed, and providing data for the store to forward to the load.

After the store instruction is committed, the store queue will move the data in it to the [committed store buffer](./lsq/committed_store_buffer.md). The committed store buffer will merge the store write requests in units of cache lines, and write the merged multiple store write requests to the first-level data cache together when it is nearly full.

The [first-level data cache](../memory/dcache/dcache.md) exposes two 64-bit read ports and a write port with the same width as the first-level data cache line to the in-core memory access component, as well as a data refill port. The width of the data refill port is determined by the bus width between the data cache and the l2 cache. Currently, the data refill port width of the Nanhu architecture is 256 bits. The first-level data cache uses the TileLink bus protocol.

[MMU](./mmu/mmu.md), also known as the Memory Management Unit, is mainly responsible for translating virtual addresses into physical addresses in the processor, and then using this physical address to access memory. At the same time, permission checks are also performed, such as whether it is writable or executable. Xiangshan's [MMU](./mmu/mmu.md) includes [TLB](./mmu/tlb.md), [L2TLB](./mmu/l2tlb.md), [Repeater](./mmu/mmu.md#repeater), [PMP & PMA](./mmu/pmp_pma.md) and other components.

Multiple components work together to implement the out-of-order memory access mechanism of Xiangshan processors, see the [Memory Access Mechanism Introduction](./mechanism.md) section. In addition, the implementation of [Memory Access Violation Prediction](./mdp/mdp.md) is also described here.

!!! note
The actual Verilog code delivered to the backend involves the replacement of some code on the critical path.

The latest memory access unit design can refer to this report: [Introduction to Xiangshan Memory Access Unit Microstructure (English)](https://github.com/OpenXiangShan/XiangShan-doc/blob/main/slides/20230426-LSU_of_Xiangshan_Processor.pdf).
