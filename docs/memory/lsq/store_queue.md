# Cola de la tienda

Este capítulo presenta el diseño de la cola de almacenamiento de la arquitectura Nanhu del procesador Xiangshan.

La cola de almacenamiento de la arquitectura de Southlake es una cola circular de 64 elementos. Cada ciclo tiene como máximo:

* Recibir 4 instrucciones del despacho
* Recibir información de dirección y control para 2 instrucciones desde la tubería de direcciones de la tienda
* Recibir datos para 2 instrucciones desde la tubería de datos de la tienda
* Escribe los datos de 2 instrucciones en el búfer de almacenamiento comprometido
* Proporcionar resultados de reenvío de almacenamiento a carga para 2 canales de carga

## Cola de la tienda El contenido de cada artículo

Cada elemento de la cola de la tienda contiene la siguiente información:

* Dirección física
* Dirección virtual
* datos
* Máscara de datos válida
* Bits de estado

Bits de estado | Descripción
-|-
asignado|El artículo ha sido asignado por envío
El comando addrvalid|store ha obtenido la dirección requerida
El comando datavalid|store ha obtenido los datos requeridos
El comando commited|store ha sido confirmado
mmio|Esta instrucción de almacenamiento accede al espacio de direcciones mmio
pendiente|La instrucción de almacenamiento accede al espacio de direcciones mmio y se pospone la ejecución. Esperando a que la instrucción se convierta en la última instrucción en el ROB

## Asignación y puesta en cola de la tienda

La instrucción de almacenamiento ingresa a la cola de almacenamiento en dos pasos: la asignación temprana de enqPtr y la escritura real en la cola de almacenamiento. El flujo de procesamiento del almacenamiento aquí es similar al de la carga, consulte [cargar cola en cola](./load_queue Sección .md#load-queue- enqueue).

## Problema de emisión de puntero Ptr

`issuePtr` indica que los datos y la dirección de la instrucción de almacenamiento anterior a este sqIdx están listos. La cola de almacenamiento proporciona este puntero para ayudar a la estación de reserva a programar las instrucciones de carga que dependen de la instrucción de almacenamiento anterior. issuePtr es un puntero impreciso (intente Sea lo más preciso posible (pero no obligatorio).

## Actualización de la tienda Cola de la tienda

### Dirección de la tienda

Una instrucción de almacenamiento actualizará la cola de almacenamiento en la etapa de almacenamiento 1/etapa 2 de la secuencia de comandos sta. Entre ellas, la etapa de almacenamiento 1 actualiza la dirección y la mayoría de los bits de estado. Si no hay errores de TLB, se activa el indicador `addrvalid`. Conjunto. Actualizaciones de la etapa 2. La razón por la cual los indicadores `mmio` y `pendiente` se actualizan en la etapa 2 es que el tiempo que tarda PMA en verificar si la instrucción está ubicada en el área MMIO es ajustado.

### Almacenar datos

Los datos de la tienda enviados desde una estación de reserva se escribirán en la cola de la tienda inmediatamente. El indicador `datavalid` se activará durante la escritura.

<!-- ### Caso especial

* (1) Para una instrucción mmio con excepciones, debemos marcarla como addrvalid (de esta manera disparará una excepción cuando llegue a la cabeza de ROB) en lugar de pendiente para evitar enviarlas a un nivel inferior.
* (2) Para una instrucción mmio sin excepciones, la marcamos como pendiente. Cuando la instrucción llega a la cabecera de ROB, StoreQueue la envía al canal de descache. Al recibir la respuesta, StoreQueue vuelve a escribir la instrucción a través del árbitro con unidades de almacenamiento. Más tarde, confirmar normalmente. -->

## Mecanismo relacionado con el envío de la tienda

Después de enviar el comando, rob genera una señal `scommit` según la cantidad de comandos de almacenamiento enviados, notificando a la cola de almacenamiento que esta cantidad de comandos de almacenamiento se han enviado correctamente. Dado que la cola de almacenamiento está lejos de ROB, la cola de almacenamiento En realidad, utiliza scommit para actualizar el estado interno en ROB. Dos segundos después de que se ejecuta el comando de confirmación.

La cola de almacenamiento actualizará el indicador "comprometido" de la instrucción confirmada a verdadero para indicar que se ha confirmado y se puede escribir en el búfer.

## La tienda escribe en Sbuffer

Las instrucciones de almacenamiento enviadas no se cancelarán, se leerán desde la cola de almacenamiento y se escribirán en el búfer en orden.

A medida que la cola de almacenamiento continúa creciendo, el tiempo para leer datos de la cola de almacenamiento se hace cada vez más largo. Al mismo tiempo, el búfer necesita realizar una verificación al recibir datos escritos desde la cola de almacenamiento. La lógica de esta verificación es muy largo (consulte [sbuffer](../lsq/committed_store_buffer.md#) para obtener la descripción de las escrituras de sbuffer). Por cuestiones de tiempo, la arquitectura de Southlake utilizará un ciclo completo para leer datos de la cola de almacenamiento. En el siguiente ciclo, La cola de almacenamiento intentará escribir los datos leídos en el ciclo anterior en el búfer.

Antes de que los datos de la tienda se escriban en el búfer (lo que significa que el búfer puede proporcionar el resultado del reenvío de la tienda a la carga), este elemento en la cola de la tienda seguirá siendo válido para garantizar que la tienda a la carga pueda obtener correctamente el resultado de la tienda.

## Almacenar para cargar hacia adelante

Cuando la instrucción de carga realiza una consulta de almacenamiento a carga hacia adelante, el búfer proporcionará a la instrucción de carga los datos que se almacenaron antes de esta instrucción pero que no se escribieron en el búfer. En el [mecanismo de avance](../mechanism.md# actual tienda-para-cargar-adelante), la entrega hacia adelante desde la cola de la tienda utiliza la entrega hacia adelante de dirección virtual y la verificación de dirección real.

### Generar resultados de pases hacia adelante

<!-- Comparando deqPtr (deqPtr) y forward.sqIdx, tenemos dos casos:

* (1) si tienen la misma bandera, necesitamos verificar range(tail, sqIdx)
* (2) si tienen diferentes indicadores, necesitamos verificar range(tail, LoadQueueSize) y range(0, sqIdx) -->

El proceso de generación de **datos** para la entrega posterior es el siguiente: utilice la consulta de dirección real para generar el vector de coincidencia de dirección real. Luego, utilice el vector de coincidencia para generar el resultado de entrega posterior de cada byte de la cola de almacenamiento. Los datos de entrega estarán en el frente. El siguiente ritmo generado por la solicitud de entrega se devuelve al canal de carga.

### Dirección válida Datos no válidos

Cuando la lógica de avance intenta pasar los datos de la tienda a la carga posterior, puede suceder que la dirección de la tienda esté lista, pero los datos no. Cuando la lógica de avance detecta esta situación, notificará a la etapa 1 de la tubería de carga para informar el resultado. La [lógica relacionada en la canalización de carga](../fu/load_pipeline.md#store-data-invalid) es responsable del procesamiento posterior.

### Comprobación de corrección de reenvío

Para garantizar que el resultado del reenvío generado mediante el uso de la dirección virtual sea el resultado correcto, la cola de almacenamiento realizará verificaciones relacionadas con el reenvío de direcciones virtuales: una vez que se encuentre que los resultados coincidentes de las direcciones virtuales y reales son diferentes durante el reenvío, si Si los resultados son inconsistentes, la información inconsistente se enviará nuevamente a la cola de carga. Canalización. La canalización de carga maneja las fallas de reenvío.

## Mecanismo relacionado con la redirección de instrucciones

Similar a la cola de carga. Consulte [lógica de redirección en la cola de carga](../lsq/load_queue.md#redirect).

## MMIO (acceso a memoria no almacenada en caché)

El mecanismo de acceso a la memoria MMIO de la cola de almacenamiento es básicamente el mismo que el de la cola de carga.

## Vaciar la cola de la tienda

La cola de almacenamiento en sí no tiene lógica de vaciado, pero emitirá señales de vacío y de lleno al exterior. Cuando sea necesario vaciar la cola de almacenamiento, el búfer entrará en estado de actualización y escribirá todos los datos que contiene en el caché de datos. hasta que tanto la cola de almacenamiento como el búfer no contengan datos válidos. Durante este proceso, la cola de almacenamiento escribirá continuamente los datos de las instrucciones de almacenamiento enviadas en el búfer.
