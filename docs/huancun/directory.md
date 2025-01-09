# Directory Design

This chapter will introduce the design of the directory in huancun. The "directory" referred to in this chapter is broad, including metadata and tags.

huancun is a non-inclusive cache based on the directory structure, and was inspired by NCID[^ncid] during the design process. Between multi-level caches, data is non-inclusive, while the directory is inclusive. In terms of structural organization, huancun stores the directory of upper-level data and the directory of local data separately, and the structures of the two are similar. The directory is built using SRAM, with Set as the index, and each Set contains Way data.

Each item in the local data directory stores the following information:

* state: stores the permissions of the data block (one of Tip/Trunk/Branch/Invalid)

* dirty: indicates whether the data block is dirty

* clientStates: stores the permissions of the data block in the upper layer (only meaningful when the block is not Invalid)

* prefetch: indicates whether the data block is prefetched

Each item in the upper data directory stores the following information:

* state: stores the permissions of the upper data block

* alias: stores the last bit of the virtual address of the upper data block (i.e. alias bit, see [Cache alias problem](./cache_alias.md) for details)

## Directory reading

When the MSHR Alloc module allocates a request into the MSHR, it will also initiate a read request to the directory at the same time. Read the metadata and tags of the upper and local layers in parallel, determine whether it hits based on the tag comparison, and pass the metadata of the corresponding path to the corresponding MSHR based on the hit situation.

When the directory hits, the path passed is the hit path; when the directory misses, the path passed is the path obtained according to the replacement algorithm. The replacement algorithm can be flexibly configured. The local data directory of the Nanhu version uses the PLRU replacement algorithm, and the upper data directory uses the random replacement algorithm.

## Directory write

When the MHSR transaction processing is nearing the end, it is usually necessary to write the directory to update the status, tag and other information. The directory has 4 write request ports, which receive the write of local metadata, upper metadata, local tag and upper tag respectively. The priority of the write request is higher than the read request.

In terms of connection relationship, the above 4 write requests of all MSHRs are arbitrated separately in the directory. Among them, the arbitration priority relationship is carefully designed to avoid request nesting errors or deadlocks.

## Common problems and design considerations

#### The directory already has upper metadata, why is there clientStates in the local metadata?

* When a request, such as Acquire BLOCK_A, misses in the local directory, the directory will select a path to pass to MSHR according to the replacement algorithm. At this time, this path may not be invalid, but has a data block BLOCK_B. In order to complete this Acquire request, we need to know the status information of BLOCK_B in the upper layer. In order to avoid reading the directory twice, we will store an additional clientStates in the local metadata.

#### Will there be read-write competition risks in the directory?

* Our MSHR is blocked according to Set, and new requests will be allowed to enter and read the directory only when MSHR is released, so there will be no read-write competition risks.

[^ncid]: Zhao, Li, et al. "NCID: a non-inclusive cache, inclusive directory architecture for flexible and efficient cache hierarchies." *Proceedings of the 7th ACM international conference on Computing frontiers*. 2010.
