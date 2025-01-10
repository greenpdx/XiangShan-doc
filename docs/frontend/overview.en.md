# Front-end architecture document
This chapter describes the overall architecture of the Xiangshan processor front-end. The module relationship and data path of the front-end are shown in the figure below:

![frontend](../figs/frontend/frontend.png)

The Nanhu architecture adopts an instruction fetch architecture that decouples branch prediction and instruction cache. The branch prediction unit provides instruction fetch requests and writes them into a queue, which sends them to the instruction fetch unit and into the instruction cache.
The fetched instruction code is pre-decoded to preliminarily check the branch prediction errors and flush the prediction pipeline in time. The checked instructions are sent to the instruction buffer and passed to the decoding module, finally forming the back-end instruction supply.

This chapter includes the following parts:

* [Branch prediction](bp.md)
* [Instruction target queue](ftq.md)
* [Instruction fetch unit](ifu.md)
* [Instruction cache](icache.md)
* [Decode unit](decode.md)

!!! note "About abbreviations"
Some terms will appear in the document in abbreviated form (there will be an underline below them). You can see the full name of the abbreviation by hovering the mouse over it.
