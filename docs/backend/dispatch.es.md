# Despacho

En el procesador Xiangshan, la lógica de despacho consta en realidad de dos etapas de canalización. La primera etapa, Despacho, es responsable de clasificar las instrucciones y enviarlas a tres tipos de colas de despacho: de punto fijo, de punto flotante y de acceso a memoria. La segunda etapa, Despacho, es responsable de clasificar las instrucciones y enviarlas a tres tipos de colas de despacho: de punto fijo, de punto flotante y de acceso a memoria. La etapa `Dispatch2Rs` es responsable de despachar instrucciones del tipo correspondiente a diferentes estaciones de reserva según los diferentes tipos de operación.

En la actualidad, la lógica general de la etapa de despacho es larga y compleja, y todavía hay mucho margen de optimización.

## Despacho de primer nivel

El código fuente del despacho de primer nivel está en `Dispatch.scala`, que incluye el juicio de diferentes tipos de instrucciones, el juicio de si cada instrucción puede ingresar al siguiente nivel y la configuración de BusyTable (la instrucción se colocará en este nivel de tubería). Su estado de registro de destino está establecido como no válido).

Es importante tener en cuenta que, por consideraciones de tiempo, hemos simplificado las condiciones bajo las cuales las instrucciones pueden continuar al siguiente nivel. Actualmente, las instrucciones pueden ingresar a la siguiente etapa si y solo si todos los recursos son suficientes (la cola de despacho tiene suficientes entradas vacías, ROB tiene suficientes entradas vacías, etc.). Es decir, las instrucciones de punto fijo pueden bloquearse porque la cola de despacho de punto flotante está llena.

## Cola de despacho

La cola de despacho es el puente entre el primer nivel y el segundo nivel, en el que se almacenan algunas instrucciones. Cuando ocurre una solicitud de redirección, como una predicción de bifurcación, existe una cierta posibilidad de que las instrucciones en esta cola se eliminen o no, por lo que la lógica de la cola también incluye una sentencia de cancelación para `robIdx`.

## Segundo nivel Dispatch2Rs

El despacho de segundo nivel es responsable de enrutar las instrucciones y enviarlas a diferentes estaciones de reserva en función de los diferentes tipos de instrucciones y los tipos de instrucciones aceptables para las diferentes estaciones de reserva. Actualmente, hemos implementado varias estrategias de despacho simples y el sistema parametrizado no ha sido probado completamente.
