# Integer unit

## ALU

Xiangshan's ALU supports addition, subtraction, logic, branch, partial bit extension and other instructions.

## MUL

Xiangshan's multiplier defaults to a 3-stage pipeline Wallace tree multiplier, which can also be modified to a multiplier directly implemented by `*` through configuration, and then
register retiming is used to optimize timing.

## DIV

Xiangshan uses an SRT16 fixed-point divider[^1], which operates 4 bits per cycle and processes two beats before and after the division cycle.

## MISC: CSR/JUMP

Xiangshan's CSR implementation supports the riscv-priv-1.12 version specification.

Due to the special operand requirements of the JUMP instruction, we implemented the operation of the JUMP instruction in the MISC unit.

[^1]: E. Antelo, T. Lang, P. Montuschi and A. Nannarelli, "Digit-recurrence dividers with reduced logical depth", IEEE Trans. Comput., vol. 54, no. 7, pp. 837-851, Jul. 2005.
