# MSHR Design

This chapter introduces the design of non-inclusive MSHR (Miss Status Holding Registers) in huancun.

When huancun receives an Acquire/Release request from the upper cache, or a Probe request from the lower cache, it will assign an MSHR to the request, and obtain the permission information of the address in the cache at this level and above by reading the directory. MSHR decides based on this information:

* [Permission control](#meta_control): how to update the permissions in the self directory/client directory;

* [Request control](#request_control): whether to send sub-requests to the upper and lower caches and wait for the responses of these sub-requests;

<!--
TODO
* How to deal with nested requests, etc.
-->
The following will <b>take L2 Cache as an example</b> to introduce the design of huancun MSHR from these aspects.

<h2 id=meta_control>Permission control</h2>

The data in huancun is stored non-inclusively, but the directory strictly adopts the inclusion strategy, so MSHR can obtain all the permission information of the request address in ICache, DCache and L2 Cache, thereby controlling the change of the address permission. Each address in XiangShan's cache system follows the rules of the TileLink consistency tree. Each block has four states in each layer of the cache system: N (None), B (Branch), T (Trunk), and TT (TIP). The first three correspond to no permission, read-only, and readable and writable. The consistency tree grows from bottom to top in the order of memory, L3, L2, and L1. The memory, as the root node, has read and writable permissions. The permissions of the child nodes in each layer cannot exceed the permissions of the parent node. Among them, TT represents the leaf node on the branch with T permission, which means that the upper layer of the node has only N or B permissions. On the contrary, the node with T permission but not TT permission means that there must be T/TT permission nodes in the upper layer. For detailed rules, please refer to Chapter 9.1 of the TileLink manual.

MSHR updates the dirty bit, permission domain, and clientStates domain of the self directory according to the type of request and the result of reading the directory. ClientStates indicates what the permission of this address is in the upper L1 Cache if the address has permission in the current cache (B or above). In addition, MSHR also updates the client directory corresponding to ICache and DCache, including the permission domain and alias domain (which will be introduced in the chapter [Cache Alias ​​Problem](./cache_alias.md)).

The following two examples briefly explain how MSHR determines the modification of permissions and why these domains are needed in the design of the directory.

1. If DCache has permission for address X, but ICache does not, ICache requests permission for X from L2. At this time, L2 needs to determine whether L2 has block X, whether DCache has block X, and if so, whether the permissions are sufficient. Based on this information, MSHR will decide whether to return data directly to ICache, or to ask DCache for data (Probe) and then return it to ICache, or to Acquire the block from L3 and then forward it to ICache, so we need to maintain the permission bits in the self directory and client directory respectively.

2. If DCache has the permission to address X, but L2 does not, then DCache replaces X and sends a request to release address X downward. After receiving the request, L2 allocates an MSHR. At the same time, we hope that L2 can save block X released by DCache. If replacement is required, L2 must also release replacement block Y to L3 according to the replacement strategy. In this example, we need to know whether Y is dirty data in L2, which involves whether data needs to be brought when releasing Y to L3, so the dirty bit is needed in the self directory; when replacing Y out of L2, we need to know what kind of param the Release should bring, so the clientStates domain needs to be maintained in the self directory to determine the permissions of the upper node at address Y.

For more details about directory, please read [Directory Design](./directory.md).

<h2 id=request_control>Request Control</h2>

MSHR needs to determine which sub-requests need to be completed based on the content of the request and the result of reading the directory, including whether to acquire or release downward, whether to probe upward, whether to trigger a pre-fetch, whether to modify the directory and tag, etc.; in addition to sub-requests, MSHR also needs to record which sub-requests need to wait for responses.

MSHR specifies these requests to be scheduled and responses to be waited for as events, and uses a series of status registers to record whether these events are completed. The `s_*` registers represent the requests to be scheduled, and the `w_*` registers represent the responses to be waited for. After getting the results of reading the directory, MSHR will set the events to be completed (`s_*` and `w_*` registers) to `false.B`, indicating that the request has not been sent or the response has not been received. After the event is completed, the registers will be set to `true.B`. When all events are completed, the MSHR will be released.
