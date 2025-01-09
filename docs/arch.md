# Xiangshan processor overall architecture design

The Xiangshan processor is a six-issue out-of-order structure design, currently supporting the RV64GCBK extension (the specific instruction set string is `RV64IMAFDC_zba_zbb_zbc_zbs_zbkb_zbkc_zbkx_zknd_zkne_zknh_zksed_zksh_svinval_zicbom_zicboz`). The front-end pipeline of the Xiangshan processor includes branch prediction unit, instruction fetch unit, instruction buffer and other units, and sequential instruction fetch. The back-end includes decoding, renaming, reordering buffer, reservation station, integer/floating-point register file, integer/floating-point operation unit. We separated the memory access subsystem, including two load pipelines, two store addr pipelines, two store data pipelines, and independent load queues and store queues, store buffers, etc. The cache includes ICache, DCache, L2/L3 Cache (HuanCun), TLB and prefetcher modules. The position of each part in the pipeline and the parameter configuration can be obtained from the figure below.

![Xiangshan Architecture Diagram](./figs/nanhu.png)

For detailed structural design, please refer to the introduction of the corresponding chapter.
