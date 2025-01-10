# Tubería de carga

En este capítulo se presenta el diseño del pipeline de carga de la arquitectura Nanhu del procesador Xiangshan y el flujo de procesamiento de las instrucciones de carga. El procesador Xiangshan (arquitectura Yanqi Lake) contiene dos pipelines de carga, cada uno de los cuales se divide en tres etapas de pipeline.

<!-- !!! todo
 actualizar gráfico -->
<!-- ![tubería de carga](../../figs/memblock/load-pipeline.png) -->

## División de tuberías

El proceso de ejecución de instrucciones de carga se divide en las siguientes etapas:

### Etapa 0

* Las instrucciones y operandos se leen desde la estación de reserva.
* El sumador suma el operando al valor inmediato y calcula la dirección virtual
* La dirección virtual se envía a la TLB para la consulta de TLB
* La dirección virtual se envía a la caché de datos para la indexación de etiquetas.

Etapa 1

* TLB genera dirección física
* Complete una comprobación rápida de anomalías
* Dirección virtual para indexación de datos
* La dirección física se envía a la memoria caché de datos para la comparación de etiquetas.
* La dirección virtual/física se envía a la cola de almacenamiento/búfer de almacenamiento comprometido para iniciar la operación de reenvío desde el almacenamiento hasta la carga.
* Con base en el vector de impacto devuelto por la memoria caché de datos de primer nivel y los resultados del juicio anormal preliminar, se genera una señal de activación temprana y se envía a la estación de reserva.
* Si ocurre un evento en este nivel que haría que la instrucción se vuelva a emitir desde la estación de reserva, notifique a la estación de reserva que la instrucción debe volver a emitirse (`feedbackFast`)

Etapa 2

* Comprobación completa de anomalías
* Seleccionar datos en función del caché de datos de primer nivel y los resultados devueltos por la recursión hacia adelante
* De acuerdo con los requisitos de la instrucción de carga, recorte los resultados devueltos
* Actualizar el estado del artículo correspondiente en la cola de carga
* El resultado (entero) se vuelve a escribir en el bus de datos común.
* El resultado (punto flotante) se envía al módulo de punto flotante

Etapa 3

* Actualizar el estado del elemento correspondiente en la cola de carga de acuerdo con los comentarios de dcache
* En función de la retroalimentación de dcache, se envía retroalimentación a la estación de reserva para informarle si es necesario reenviar esta instrucción (`feedbackSlow`)

!!! información
 La etapa 3 es responsable de enviar algunos resultados de inspección de la etapa 2 que se retrasan debido a razones de tiempo.

## Carga perdida

Una instrucción de carga errónea realizará las siguientes operaciones para obtener los datos requeridos. En esta sección se presentarán estos mecanismos uno por uno:

* Deshabilitar el despertar temprano de la instrucción actual
* Actualizar el estado de la cola de carga
* Asignar dcache MSHR (entrada MissQueue)
* Escuchar la recarga de dcache
*Escribe nuevamente la instrucción de carga de la señorita

En la etapa de carga 1, en función del resultado de la comparación de la etiqueta dcache, la canalización de carga puede saber si la instrucción actual falla. Cuando ocurre una falla, el bit válido de activación temprana de esta instrucción se establecerá en "falso" para deshabilitar la activación temprana. Activación de la instrucción actual. wake**.

En la etapa de carga 2, si se encuentra un error, la tubería de carga no volverá a escribir el resultado en la pila de registros y no ocupará el puerto de escritura diferida de la instrucción de carga. Al mismo tiempo, **el estado de la cola de carga* * se actualiza. Esta instrucción de carga que se perdió estará en la cola de carga que está esperando que dcache se rellene. Al mismo tiempo, dcache intentará asignar MSHR de dcache (entrada MissQueue) para esta instrucción de carga que se perdió. Debido a la lógica de asignación compleja , el resultado de la asignación no se devolverá a la cola de carga hasta el siguiente latido.

En la etapa de carga 3, **el estado de la cola de carga se actualiza nuevamente según el resultado de la asignación de MSHR de dcache**. Si la asignación de MSHR de dcache falla, se solicita a la estación de reserva que vuelva a emitir esta instrucción.

Si a esta instrucción se le asigna correctamente un MSHR de dcache, escuchará el resultado de la recarga de dcache en la cola de carga. Consulte [cola de carga: recarga de carga](../lsq/load_queue.md#load-refill).

## Repetición desde RS

La tubería de carga es **sin bloqueo**, lo que significa que, independientemente de la situación anormal que se produzca, no afectará el flujo de instrucciones válidas en la tubería. Cuando se produce una situación anormal distinta a una falla de carga y hace que la carga se Si no se completa con normalidad, la carga La canalización volverá a ejecutar esta instrucción utilizando el mecanismo [Replay From RS](../mechanism.md#Replay-From-RS). A continuación, se describen los eventos que activan la reproducción desde la reserva. mecanismo de la estación uno por uno. :

Señorita TLB

Los eventos de **falla de TLB** solicitarán la reemisión desde la estación de reserva mediante el puerto feedbackSlow. Las instrucciones de falla de TLB tienen un retraso de reemisión cuando se vuelven a emitir, y se vuelven a emitir después de que la instrucción espera en la estación de reserva hasta que finaliza el retraso. Reemisión La demora existe porque la recarga de TLB lleva tiempo. Volver a emitir instrucciones antes de que se complete la recarga de TLB provocará un error de TLB, lo cual no tiene sentido.

### Conflicto bancario

**Eventos de conflicto bancario**. Los eventos de conflicto bancario incluyen conflictos bancarios entre dos canales de carga, así como conflictos entre canales de carga y operaciones de almacenamiento que escriben líneas de caché (las operaciones de almacenamiento aquí se refieren a almacenamientos comprometidos que escriben en búferes de almacenamiento comprometidos). Actualmente, solo Permite ejecutar dos instrucciones de carga simultáneamente sin desencadenar un conflicto de bancos. En cuanto a los conflictos de carga/almacenamiento, no tenemos comprobaciones complejas debido a restricciones de tiempo. Mientras las operaciones de carga/almacenamiento actúen sobre la misma línea de caché, se considera que hay conflicto. Las retransmisiones desde un conflicto bancario utilizan el puerto feedbackFast. Las retransmisiones desde estaciones de reserva no sufren demoras. Las estaciones de reserva pueden retransmitir esta instrucción inmediatamente después de recibir una solicitud de retransmisión de conflicto bancario.

### Error de asignación de MSHR de Dcache

**Error en la asignación de MSHR de dcache**. Para conocer los motivos del error en la asignación de MSHR de dcache, consulte [dcache/MissQueue](../dcache/miss_queue.md). La retransmisión causada por un error en la asignación de MSHR de dcache utiliza el puerto feedbackSlow para emitir una solicitud de retransmisión. Retransmisión Sin demora, la estación de reserva puede volver a emitir la instrucción inmediatamente al recibir la solicitud de reemisión de falla de asignación de MSHR de dcache.

!!! nota
 Se trata de un problema de optimización del diseño y las fallas en la asignación de MSHR de dcache pueden deberse a varios motivos diferentes. Gestionar estos casos por separado beneficiará el rendimiento.

### Los datos de la tienda no son válidos

**La dirección de la tienda está lista, pero los datos no están listos**. Estas instrucciones no actualizarán la cola de carga ni volverán a escribir, pero emitirán una solicitud de reenvío a través de feedbackSlow en la etapa de carga 3 para notificar a la estación de reserva que esta instrucción está esperando. anterior Los datos de una instrucción de almacenamiento están listos**. Durante la comprobación de avance de la carga de almacenamiento, se comprobará el sqIdx del almacenamiento del que depende la carga y se enviará de vuelta a la estación de reserva a través del puerto feedbackSlow. Hay un no- Retardo fijo. La estación de reserva puede esperar hasta que se generen los datos de almacenamiento correspondientes según el sqIdx encontrado antes de volver a emitir esta instrucción de carga.

## Manejo de excepciones

Las excepciones que puede manejar la canalización de carga se dividen en dos categorías: excepciones de verificación de direcciones y excepciones de manejo de errores. Las excepciones utilizan rutas de excepción independientes y la sincronización es más flexible que la ruta de datos. Los resultados de la verificación de direcciones se generan en etapas para optimizar Para obtener más información sobre el rendimiento temporal, consulte la sección [MMU](../mmu/mmu.md).

## Procesamiento de instrucciones de precarga

Actualmente, las instrucciones de precarga de software utilizan un flujo de procesamiento similar para cargar instrucciones. Las instrucciones de precarga de software ingresan al flujo de carga como instrucciones de carga normales y, cuando se encuentra un error, se envía una solicitud a MissQueue de la caché de datos, lo que activa el acceso a la caché inferior. . ,Todas las excepciones se bloquearán durante la ejecución de la instrucción de precarga de software y no se volverán a emitir.

## El despertar temprano de Load

La estación de reserva de Nanhu admite un mecanismo de activación temprana para programar las instrucciones posteriores lo antes posible. Sin embargo, la arquitectura de Nanhu actualmente no admite el mecanismo de activación especulativa. Las instrucciones que se activan con anticipación deben poder ejecutarse normalmente. De lo contrario, se debe vaciar todo el pipeline. Si una instrucción de carga se ejecuta normalmente pero no se emite ninguna señal de activación temprana, las instrucciones subsiguientes que dependen de esta instrucción de carga se emitirán un ciclo más tarde, lo que resultará en una ligera pérdida de rendimiento.

La tubería de carga dará una señal de activación rápida a la estación de reserva en la etapa de carga 1. Debido al retraso de la línea entre MemBlock e IntBlock, esta señal está en la ruta crítica. Cuando se genera la señal de activación temprana, la carga La canalización comprobará si la instrucción se puede ejecutar con normalidad. Si la instrucción muestra signos de no poder ejecutarse con normalidad, no se reactivará con antelación.

En casos extremos, la instrucción de carga emitirá incorrectamente una activación temprana y será necesario vaciar todo el pipeline. Las condiciones para que la instrucción de carga emita incorrectamente una activación temprana son: la instrucción de carga tiene una excepción después de la etapa de carga 1 (debido a error de datos/error de bus).

## Falla de reenvío

Esta sección describe cómo la arquitectura de Southlake maneja las fallas de reenvío de direcciones virtuales. Cuando la cola de almacenamiento o el búfer de almacenamiento informa que el reenvío de direcciones virtuales ha fallado, la canalización de carga agregará una instrucción `replayInst` a esta instrucción en la etapa 2 (que requiere obtener la instrucción de la tubería de carga). Etiqueta de reenvío. La redirección se activa cuando esta instrucción llega al final de la cola ROB. Dado que la falla del reenvío de direcciones virtuales es un fenómeno muy poco frecuente, dicha redirección no tendrá un impacto significativo en el rendimiento.

## Relacionado con la depuración

El mecanismo de activación se establece en la canalización de carga. Por cuestiones de tiempo, la arquitectura de Southlake solo admite el uso de direcciones como condiciones de activación.
