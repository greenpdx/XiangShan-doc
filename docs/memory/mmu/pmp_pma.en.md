# Physical Address Protection

Physical address protection is divided into two parts: PMP (Physical Memory Protection) and PMA (Physical Memory Attribute).
The Yanqihu architecture supports hard-written PMA checks, but not PMP checks. The Nanhu architecture adds support for PMP, and also supports software readable and writable PMA implementation.

The implementation of PMP follows the RV manual, with 16 items by default, which can be parameterized (but needs to be a multiple of 8). For timing considerations, a distributed & replicated implementation method is adopted. The PMP register in the CSR unit is responsible for instructions such as CSRRW; a copy of the PMP register is available at the front-end instruction fetch, back-end memory access, and Page Walker, and the CSR write signal is pulled to ensure consistency with the PMP register in the CSR unit.

The implementation of PMA adopts a PMP-like method, using the two reserved bits of the PMP Configure register, set to atomic and cacheable, respectively: whether atomic operations are supported and whether cacheable. According to the manual, the PMP register has no initial value (empty by default). The PMA register has an initial value by default and needs to be manually set to be consistent with the platform address attribute. The PMA register uses the M-state CSR reserved register address space, 16 items, and can be parameterized.

PMP and PMA checks are queried in parallel. If one of the permissions is violated, it is an illegal operation. All physical address accesses in the core need to be checked for physical address permissions, including after ITLB and DTLB checks and before Page Walker accesses memory. According to the manual, Page Fault has a higher priority than Access Fault, but when Page Walker has an Access Fault, the page table entry is illegal. At this time, a special situation occurs in which Page Fault and Access Fault appear at the same time. Xiangshan chooses to report Access Fault, which may be inconsistent with the manual.

According to the manual, the checks of PMP and PMA should be dynamic checks, that is, they need to be translated by TLB and the translated physical address is used for physical address permission checks. However, for timing considerations, the check results of the DTLB normal page part are queried in advance and stored in the TLB entry. For this purpose, the granularity of PMP and PMA needs to be increased to 4KB.

When peripherals such as DMA access memory, physical address checks are also required, so a memory-mapped version of PMA is implemented for use by peripherals such as DMA.
