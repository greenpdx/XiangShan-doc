# Problema de lanzamiento


En la etapa de emisión de instrucciones, el módulo principal involucrado es la estación de reserva (o cola de emisión), que en el código se denomina `ReservationStation`, abreviado como RS. Las instrucciones involucradas en la fase de emisión incluyen principalmente operaciones como poner en cola, seleccionar, leer datos y sacarlos de la cola. Al mismo tiempo, la estación de reserva también es responsable de monitorear la reescritura de instrucciones y reactivar las instrucciones en espera. Actualmente, Xiangshan adopta el diseño de leer la pila de registros antes del lanzamiento.

## Módulos principales y estructuras de datos

Las principales estructuras de datos en la estación de retención incluyen el estado de instrucción `StatusArray`, el almacenamiento de instrucciones `PayloadArray` y el almacenamiento de datos `DataArray`. El módulo principal también incluye la lógica de selección `SelectPolicy`, que se utiliza para asignar entradas de tabla libres para instrucciones en cola y seleccionar instrucciones listas para su emisión.

## Únete al equipo

El comando se pone en cola a través de la interfaz `io.allocate` y los elementos correspondientes se actualizarán a `StatusArray` y `PayloadArray` en T ciclos. Los datos leídos del archivo de registro llegarán en el ciclo T + 1, por lo que se omitirán en el ciclo actual (si la instrucción seleccionada es la instrucción en cola en el ciclo anterior) o se almacenarán en `DataArray`. Dependiendo de la diferente información del comando, el estado en `StatusArray` se actualizará en consecuencia.

## elegir

En el ciclo T, el módulo `StatusArray` generará las instrucciones que se pueden activar en el ritmo actual y las ingresará al módulo `SelectPolicy` para obtener las instrucciones listas que están seleccionadas en el ritmo actual. Debido a la existencia del algoritmo AGE, en este momento se seleccionarán las instrucciones `IssueWidth + 1` y las instrucciones disponibles para emisión se determinarán en el ciclo T + 1.

## Leer datos

Actualmente, Xiangshan implementa un diseño que lee la pila de registros antes de emitir. Para este diseño, los operandos de la instrucción se almacenarán en RS una vez más y el índice es la posición de la instrucción correspondiente en RS. `DataArray` es un diseño de lectura asincrónica que lee los operandos correspondientes según la posición de RS.

Xiangshan se puede modificar fácilmente a un diseño que lee el archivo de registro después del lanzamiento, en cuyo caso el índice de los datos leídos es el número de registro físico correspondiente.

## Partida

Dequeue es el último nivel de flujo en la estación de reserva. La instrucción se selecciona en T0, T1 lee los datos y T2 completa el protocolo de enlace y los retira de la cola. La interfaz correspondiente es `io.deq`.

## Monitoreo y activación

La estación de reserva debe monitorear la señal de reescritura de la instrucción (en el diseño de lectura del archivo de registro antes de la emisión, también es necesario monitorear los datos reescritos por la instrucción) y guardar los operandos requeridos por la instrucción en el estación de reserva en consecuencia. El módulo `StatusArray` es responsable de monitorear el bus de escritura diferida y determinar qué instrucciones tienen operandos que coinciden con él. Los datos de escritura diferida de la señal y la instrucción coincidentes se enviarán al puerto de `DataArray`, y se capturarán los datos de escritura diferida.

## Optimizar la estrategia de emisión de instrucciones de multiplicación-suma de punto flotante

La unidad FMA del procesador Xiangshan admite la separación de multiplicación y suma y puede procesar multiplicación y suma de punto flotante simultáneamente. Por lo tanto, para las instrucciones de multiplicación-suma de punto flotante, implementamos la optimización de su emisión en la estación de reserva. Cuando los dos operandos de la multiplicación están listos, permitimos que se emita la instrucción y escribimos nuevamente el resultado intermedio de la multiplicación de punto flotante. A la estación de reserva. Cuando el segundo operando para la suma está listo, se emite nuevamente la instrucción y se completa toda la operación.
