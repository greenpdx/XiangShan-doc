# Memory Access Dependency Prediction

This chapter describes the overall architecture of the Xiangshan processor memory access violation prediction predictor.

Xiangshan processor predicts memory access dependency based on PC near decode. The current code supports two prediction methods:

* Load Wait Table[^21264]
* Variant of Store Sets[^storesets]

Nanhu architecture uses a memory access dependency prediction mechanism similar to Store Sets. If a load instruction is predicted to be a possible violation, the load instruction needs to wait in the reservation station until all previous stores are issued before it can be issued.

## Nanhu's memory access violation predictor

### Dependency prediction

* The overall idea is based on Store Sets, and two tables in Store Sets are implemented: SSIT and LFST

* Nanhu architecture only predicts violations caused by store addr not being ready

* A Store Set will track **multiple** inflight stores at the same time. Only when all stores here are executed, the load that depends on this Store Set will be predicted to have no dependency

* SSIT queries are performed in rename. LFST queries are performed in dispatch

### Dependency handling

* The memory access violation predictor of the Nanhu architecture uses robIdx to track the relationship between instructions
* Unlike the original Store Sets, **store instructions in the same Store Set are not required to be executed sequentially**
* The predictor will **try to predict whether the load instruction may depend on the results of multiple stores**. If the predictor believes that the load instruction does not depend on the results of multiple stores, then this load instruction can be emitted after the address of the last dependent store before this load is generated. Otherwise, this load will only be emitted after all the stores it is predicted to depend on have generated addresses.

### Predictor update

When a store-load violation occurs, the PC of the instruction that triggered the violation will be passed to the memory access violation predictor for update. The information in the violation predictor will be invalidated after each refresh interval. The refresh interval can be configured using the `slvpredctl` CSR register.

<!-- When the violation predictor is refreshed, the corresponding item will be directly invalidated. The design of adding confidence will be considered later, Only the ones with low confidence are flushed. -->

<!-- Currently, the results of normal load execution will not be fed back to the predictor for updating. The relevant design will be considered in the next version. -->

## Quote

[^21264]: Kessler, Richard E. "The alpha 21264 microprocessor." IEEE micro 19.2 (1999): 24-36.

[^storesets]: Chrysos, George Z., and Joel S. Emer. "Memory dependence prediction using store sets." ACM SIGARCH Computer Architecture News 26.3 (1998): 142-153.
