# Componentes enteros

ALU

La ALU de Xiangshan admite instrucciones como suma, resta, lógica, ramificación y extensión parcial de bits.

##MUL

El multiplicador predeterminado de Xiangshan es un multiplicador de árbol de Wallace en cadena de 3 etapas, que también se puede cambiar a un multiplicador implementado directamente por `*` a través de la configuración.
Optimice la sincronización mediante la resincronización del registro.

##División

Xiangshan utiliza el divisor de punto fijo SRT16[^1], que opera 4 bits por ciclo y procesa dos pasos antes y después del ciclo de división.

## MISC: RSE/SALTAR

La implementación de CSR de Xiangshan admite la especificación de la versión riscv-priv-1.12.

Debido a los requisitos especiales de operandos de la instrucción JUMP, implementamos la operación de la instrucción JUMP en la unidad MISC.

[^1]: E. Antelo, T. Lang, P. Montuschi y A. Nannarelli, "Divisores de recurrencia de dígitos con profundidad lógica reducida", IEEE Trans. Comput., vol. 54, núm. 7, págs. 837- 851, julio de 2005.
