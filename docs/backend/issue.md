# Issue

During the instruction emission phase, the main module involved is the reservation station (or emission queue), which is `ReservationStation` in the code, referred to as RS. The instructions involved in the emission phase mainly include enqueue, selection, data reading, dequeue and other operations. At the same time, the reservation station is also responsible for monitoring the instruction write back and waking up the waiting instructions. Xiangshan currently adopts the design of reading the register stack before emission.

## Main modules and data structures

The main data structures in the reservation station include instruction status `StatusArray`, instruction storage `PayloadArray` and data storage `DataArray`. The main module also includes the selection logic `SelectPolicy`, which is used to allocate free table entries for enqueued instructions and select ready instructions for emission.

## Enqueue

Instructions are enqueued through the interface `io.allocate`, and the corresponding items will be updated to `StatusArray` and `PayloadArray` in T cycles. The read data of the register file will arrive in the T + 1 cycle, so it will be bypassed in the current shot (if the selected instruction is the instruction queued in the previous shot) or stored in `DataArray`. According to the different information of the instruction, the status in `StatusArray` will be updated accordingly.

## Selection

In the T cycle, the `StatusArray` module will output the instructions that can be awakened in the current shot and input them to the `SelectPolicy` module to obtain the ready instructions selected in the current shot. Due to the existence of the AGE algorithm, the current shot will select `IssueWidth + 1` instructions and determine the instructions available for issuance in the T + 1 cycle.

## Read data

At present, Xiangshan implements the design of reading the register file before issuance. For such a design, the operands of the instruction will be stored in RS one more time, and the index is the position of the corresponding instruction in RS. `DataArray` is an asynchronous read design, which reads the corresponding operands according to the position of RS.

Xiangshan can be easily modified to a design that reads the register file after emission. In this case, the index of the read data is the corresponding physical register number.

## Dequeue

Dequeue is the last stage of the reservation station. The instruction is selected at T0, T1 reads the data, and T2 completes the handshake and dequeues. The corresponding interface is `io.deq`.

## Listening and wake-up

The reservation station needs to listen to the write-back signal of the instruction (in the design of reading the register file before emission, it is also necessary to listen to the data written back by the instruction), and save the operands required by the instruction in the reservation station accordingly. The `StatusArray` module is responsible for listening to the write-back bus and determining which instructions have operands that match it. The matching signal and the write-back data of the instruction will be sent to the port of `DataArray`, and the write-back data will be captured.

## Optimizing the emission strategy of floating-point multiplication and addition instructions

The FMA unit of the Xiangshan processor supports multiplication and addition separation, and can process floating-point multiplication and addition at the same time. Therefore, for floating-point multiply-add instructions, we implement their issuance optimization in the reservation station. When the two operands of the multiplication are ready, we allow the instruction to be issued and write the intermediate result of the floating-point multiplication back to the reservation station. When the second operand of the addition is ready, the instruction is issued again and completes the entire operation.
