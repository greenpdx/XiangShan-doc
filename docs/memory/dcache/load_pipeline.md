# Load Pipeline

This chapter describes the implementation of the Xiangshan processor DCache Load pipeline.

The DCache load pipeline corresponds to the load unit load pipeline in each pipeline stage and flows synchronously. The two can be regarded as the same pipeline logically. The DCache load pipeline is **non-blocking**. Once the dcache pipeline stage 0 receives a load instruction, the load instruction will not be blocked. However, in the event of a conflict, the dcache pipeline stage 0 may refuse to receive the load instruction.

## DCache Load Pipeline Division

### Stage 0

* Receive the virtual address calculated by the load pipeline
* ​​Use the virtual address to query the tag
* Use the virtual address to query the meta

### Stage 1

* Get the tag query result
* Get the meta query result
* Perform tag matching
* Determine whether the dcache access is a hit
* Check the tag error
* Use the physical address to query the data
* Get PLRU information, select replacement way
* Check bank conflict
* Generate fast wake-up signal

### Stage 2

* Update dcache PLRU
* Get data query result
* If miss, try to allocate MSHR (miss queue) item
* Check data error
