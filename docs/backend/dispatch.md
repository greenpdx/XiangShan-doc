# Dispatch

In the Xiangshan processor, the dispatch logic actually includes two pipeline stages. The first stage `Dispatch` is responsible for classifying instructions and sending them to three types of dispatch queues (Dispatch Queue) of fixed-point, floating-point and memory access. The second stage `Dispatch2Rs` is responsible for further dispatching the corresponding types of instructions to different reservation stations according to different operation types.

At present, the overall logic of the dispatch stage is long and complex, and there is still a lot of room for optimization.

## First-level Dispatch

The source code of the first-level dispatch is in `Dispatch.scala`, which includes the judgment of different instruction types, the judgment of whether each instruction can enter the next stage, and the setting of BusyTable (the instruction will set its destination register status to invalid at this pipeline stage).

It should be noted that for timing considerations, we have simplified the conditions for instructions to continue to enter the next stage. At present, instructions can enter the next stage if and only if all resources are sufficient (there are enough empty items in the dispatch queue, enough empty items in the ROB, etc.). That is to say, fixed-point instructions may be blocked because the floating-point dispatch queue is full.

## Dispatch Queue

The dispatch queue is a bridge between the first level and the second level, in which some instructions are stored. When redirection requests such as branch prediction occur, there is a certain possibility that the instructions in this queue will be flushed/not flushed, so the queue logic also includes the cancellation judgment of `robIdx`.

## Second level Dispatch2Rs

The second level dispatch is responsible for routing instructions according to different instruction types and instruction types acceptable to different reservation stations, and sending them to different reservation stations. At present, we have implemented several simple dispatch strategies, and the parameterized system has not been fully tested.。
