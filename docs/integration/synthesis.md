# Logic Synthesis Guide

This section describes the necessary information to synthesize the NANHU design.

## Clock and Reset &nbsp; Clock and Reset

Please refer to the SoC Section for clock and reset configurations.

## L3 Cache SRAM Timing
The L3 cache in NANHU is by default configured to be accessed (read or write) once during every two cycles.

* SRAM Writing &nbsp; SRAM write

![](../figs/integration/sram_write.png)

* SRAM Reading &nbsp; SRAM read

There are two possible cases for SRAM reading, depending on the relation between read latency of SRAM itself and the frequency of processor. Although the behavior of SRAM under two situation is different, both of them are correct since the L3 Cache only uses the rdata after two cycles.

Case 1: Latency(SRAM) < CycleTime(processor) &nbsp; SRAM latency < processor cycle

![](../figs/integration/sram_read_1.png)

Case 2: Latency(SRAM) > CycleTime(processor) &nbsp; SRAM latency > processor cycle

![](../figs/integration/sram_read_2.png)

* SRAM Read + Write<br>SRAM read and write

Case 1: Latency(SRAM) < CycleTime(processor) &nbsp; SRAM latency < processor cycle

![](../figs/integration/sram_rw_1.png)

Case 2: Latency(SRAM) > CycleTime(processor) &nbsp; SRAM latency > Processor cycles

![](../figs/integration/sram_rw_2.png)
