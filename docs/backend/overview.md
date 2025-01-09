# Tubería de back-end

## Diseño general

El back end del pipeline del procesador es responsable del cambio de nombre y la ejecución fuera de orden de las instrucciones. Como se muestra en la figura a continuación, el backend del procesador Xiangshan se puede dividir en cuatro partes: CtrlBlock, IntBlock, FloatBlock y Memblock. CtrlBlock es responsable de decodificar, renombrar y enviar instrucciones, mientras que IntBlock, FloatBlock y MemBlock son responsables de las instrucciones de números enteros. , punto flotante y acceso a memoria, respectivamente. Ejecución de instrucciones fuera de orden.

<figura rebajada>
 ![bpu](../figs/backend/backend.svg){ ancho="800" }
 <figcaption>Diagrama de estructura del backend</figcaption>
</figura>

En estos módulos hay muchos parámetros configurables. En el código actual, los cuatro bloques anteriores utilizan las siguientes configuraciones de forma predeterminada:

* CtrlBloquear

 * Ancho de decodificación/renombrado/envío = 6

 * Leer el archivo de registro antes de transmitir

* Bloque Int

 * 192 registros físicos

 * 4 * ALU + 2 * MUL/DIV + 1 * CSR/JMP

* Bloque flotante

 * 192 registros físicos

 * 4 * FMAC + 2 * FMISC

* Bloqueo de memoria

 * 2 * CARGAR + 2 * ALMACENAR (ALMACENAR se divide en datos y dirección para un cálculo independiente)

## Implementación del código

En la actualidad, el código backend de Xiangshan está escrito de una manera relativamente básica y la configurabilidad del diseño se logra a través de la transmisión de parametrización manual. En el futuro, planeamos reescribir el código usando Diplomacy para lograr un diseño parametrizado más claro.

En la versión de código de Xiangshan Nanhu, la estación de reserva, la unidad funcional y la pila de registros se envuelven en ExuBlock, que contiene dos partes: Scheduler y FUBlock. El primero contiene nuevamente la estación de reserva, la pila de registros y BusyTable, mientras que el segundo contiene las funciones correspondientes. unidad. Estos niveles de empaquetado son sólo para pasar parámetros y cableado, y hay muy poco código lógico efectivo.

El ExuBlock de punto fijo incluye (1) estaciones de reserva para ALU, MUL/DIV y MISC; (2) estaciones de reserva para LOAD, STORE DATA y STORE ADDR; y (3) unidades funcionales ALU, MUL/DIV y MISC. .

El ExuBlock de punto flotante incluye (1) estaciones de reserva FMA y FMISC y (2) unidades funcionales FMA y FMISC.

El MemBlock de acceso a la memoria incluye el diseño del subsistema de acceso a la memoria.
