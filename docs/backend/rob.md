# Re-Order Buffer (ROB)

In an out-of-order processor, the role of ROB is to order instructions so that the results of normal program execution can be preserved. According to the execution process of an instruction, ROB will affect the dispatch, writeback, and submission process of the instruction in sequence, and the instruction may be flushed at any time.

## Dispatch

In the Dispatch stage, the instruction will be assigned a ROB table entry, and some information to be saved `RobCommitInfo` will be stored in the ROB, such as the instruction renaming information, type information, corresponding ftq pointer, etc. The width of the ROB queue is consistent with the width of the renaming.

After the instruction enters the ROB, some status bits will be updated, such as `valid`, `writebacked`, `interrupt_safe` (for the sake of simplifying the design, the information of whether the memory access instruction is MMIO will not be passed to the ROB. They will access the memory before submission, so we currently simplify the design to avoid memory access instructions triggering interrupts), etc.

## Writeback

After the instruction is executed, the ROB will be notified that the corresponding operation has been completed, and the ROB will set the `writebacked` flag to `true`.

## Submit

In each clock cycle, the ROB will check whether the instructions at the head of the queue can be submitted normally, and submit as many instructions as possible through the `io.commits` interface.

For instructions with exceptions, their submission will be blocked, and exception information will be sent out through the `io.exception` interface.

## Cancel and rollback

During the execution of an instruction, if branch prediction errors, memory access violations, etc. occur, the instruction and subsequent instructions may need to be flushed. In this case, the ROB will receive cancellation information through the `io.redirect` port, and determine which part of the instructions need to be canceled based on the cancellation information. For canceled instructions, ROB will use the rollback mechanism to restore renaming and other information through the `io.commits` port. At this time, `io.commits.isWalk` will be set to `true`.
