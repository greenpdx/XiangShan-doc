# Implementation of atomic instructions

This chapter introduces the atomic instruction processing flow of the Xiangshan processor Nanhu architecture.

!!! note
The processing flow of atomic instructions may change in the next version

## Execution flow of atomic instructions

The atomic instructions of the Nanhu architecture are controlled by a separate state machine. An atomic instruction will go through the following states:

* s_invalid: state machine idle
* s_tlb: atomic instruction issues TLB query request
* s_pm: complete TLB query and address exception check
* s_flush_sbuffer_req: issue sbuffer/sq flush request
* s_flush_sbuffer_resp: wait for sbuffer/sq flush request to complete
* s_cache_req: initiate atomic instruction operation to dcache
* s_cache_resp: wait for dcache to return atomic instruction result
* s_finish: write result back

For store instruction, the processing flow will start only when its addr and data are ready.

## dcache Support for atomic instructions

Currently, all atomic instructions of Xiangshan (Nanhu architecture) are processed in dcache. dcache uses the main pipeline (MainPipe) to execute atomic instructions. If MainPipe finds an atomic instruction miss, it will send a request to MissQueue. MissQueue obtains data from l2 cache and then executes the atomic instruction. If MissQueue is full when the atomic instruction is executed, it will continue to try to resend the atomic instruction until MissQueue receives the miss refill request of the atomic instruction. Dcache uses one `AtomicsReplayEntry` to schedule atomic instructions.

### Special support for lr/sc

RISC-V spec defines constrained LR/SC loops (see Volume I: RISC-V Unprivileged ISA: 8.3 Eventual Success of Store-Conditional Instructions). Inspired by Rocket, Xiangshan provides a similar mechanism to handle constrained LR/SC loops.

After the lr instruction is executed, the processor will enter the following three states in sequence:

1. Block all probe requests to the current cacheline of the current core and block the execution of subsequent lr for a period of time (`lrscCycles` - `LRSCBackOff` cycles)

1. Continue to block the execution of subsequent lr, but allow probe requests for this cacheline to be executed for a period of time (`LRSCBackOff` cycles)

1. Restore to normal state

For Xiangshan (Nanhu), the constrained LR/SC loop (up to 16 instructions) must be executed within the `lrscCycles` - `LRSCBackOff` cycle. During this period, the probe operation to the address in the reservation set set by LR will be blocked. The current hart can execute a successful SC without being interrupted. The subsequent lr will not be executed in the subsequent `LRSCBackOff` cycle to prevent two cores from executing lr at the same time, resulting in both of them being unable to accept the probe Request, neither can get the permission and thus get stuck. After adding this backoff phase, probe requests from other cores can obtain cacheline permissions.

### Support for amo instructions

dcache will perform the calculation operations defined by amo instructions in mainpipe. See [dcache mainpipe](../dcache/main_pipe.md#stage-2) section.

## Exception handling of atomic instructions

When the address check mechanism detects an exception, the atomic instruction state machine will directly enter `s_finish` from `s_pm`, and will not trigger the store queue and sbuffer refresh operations, and will not generate actual memory access requests. dcache will not perceive atomic instructions.

## Debug related

The trigger of atomic instructions only supports the use of vaddr trigger.
