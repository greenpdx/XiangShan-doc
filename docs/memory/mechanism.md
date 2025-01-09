# Mecanismo de acceso a memoria fuera de orden

Este capítulo presenta el mecanismo clave para implementar el acceso a la memoria fuera de orden en el procesador Xiangshan.

## Cargar golpe

Cuando llega una instrucción de carga, esta pasará por 3 etapas después de ser emitida desde la estación de reserva:

* Etapa 0: Calcular dirección, leer TLB, leer etiqueta dcache
* Etapa 1: Leer datos de dcache
* Etapa 2: Obtener los resultados de la lectura, seleccionarlos y volver a escribirlos

Después de la etapa 2, hay una etapa adicional que maneja las actualizaciones de estado que la etapa 2 no tuvo tiempo de completar. Para obtener detalles sobre las operaciones realizadas por cada etapa, consulte la [canalización de carga](./fu/load_pipeline.md) detalles.

## Sta y Std

La parte de cálculo de dirección de la instrucción de almacenamiento pasará por 4 etapas después de que se emita desde la estación de reserva. Consulte [sta pipeline](./fu/store_pipeline.md#Sta-Pipeline) para obtener más detalles:

* Etapa 0: Calcular dirección, leer TLB
* Etapa 1: la dirección y otra información de control se escriben en la cola de almacenamiento y comienza la verificación de violaciones.
* Etapa 2: Comprobación de infracciones
* Etapa 3: Comprobación de infracciones, que permite enviar la instrucción de la tienda.

La parte de cálculo de datos de la instrucción de almacenamiento moverá directamente los datos desde la estación de reserva a la cola de almacenamiento después de que se emitan desde la estación de reserva, consulte [std pipeline](./fu/store_pipeline.md#Std-Pipeline).

Para obtener detalles sobre las operaciones realizadas en cada etapa, consulte la introducción detallada de [canalización de carga](./fu/load_pipeline.md).

## Manejo de errores de carga

Consulte [Error de carga](./fu/load_pipeline.md#Load-Miss).

## Repetición desde RS

Esta sección describe el mecanismo mediante el cual las instrucciones de carga y las operaciones del personal se reproducen desde las estaciones de reserva.

Tomemos como ejemplo las instrucciones de carga. Cuando se produzcan determinados eventos, volveremos a emitir estas instrucciones de carga desde la estación de reserva:

*TLB señorita
* L1 DCache MSHR completo
* Conflicto bancario DCache
* Al reenviar la dirección coincide pero los datos no están listos (Datos no válidos)

Las características comunes de estos eventos son:

* Ocurre con poca frecuencia (en comparación con las instrucciones de acceso a la memoria normales)
* Cuando ocurren estos eventos, las instrucciones de acceso a la memoria no se pueden ejecutar normalmente.
* Si la misma instrucción de acceso a la memoria se ejecuta nuevamente después de un período de tiempo, estos eventos no ocurrirán.
 * Por ejemplo, el evento de pérdida de TLB desaparecerá después de que PTW complete la recarga de TLB.

El objetivo del mecanismo de reemisión desde la estación de reserva es dejar que estas instrucciones esperen un tiempo en la pila de reserva y volver a ejecutarlas después de un cierto período de tiempo. La implementación de este mecanismo es la siguiente: después de que se emite una instrucción desde la estación de reserva, El acceso a la memoria RS, aún debe conservarse en el RS, cuando la instrucción de acceso a la memoria sale de la tubería, le informará al RS si es necesario volver a emitirla desde la estación de reserva. La instrucción que debe volver a emitirse desde la estación de reserva La estación de reserva continuará esperando en el RS para ser reemitida después de un cierto intervalo de tiempo.

Actualmente, hay dos puertos en la tubería de carga que proporcionan retroalimentación a la estación de reserva sobre si es necesario volver a emitir instrucciones. Estos puertos se encuentran en la etapa de carga 1 (`feedbackFast`) y la etapa de carga 3 (`feedbackSlow`). Las instrucciones que Las solicitudes de reenvío que se puedan detectar en la etapa 1 y que deban reenviarse se enviarán de vuelta a la estación de reserva a través del puerto `feedbackFast` de la etapa de carga 1. Las solicitudes de reenvío que solo se puedan detectar en la etapa de carga 2 se enviarán de vuelta a la estación de reserva a través de El puerto `feedbackFast` de la etapa de carga 3. El puerto feedbackSlow se utiliza para enviar información a la estación de reserva. Los dos puertos están diseñados para permitir que la estación de reserva vuelva a enviar algunas instrucciones que deben enviarse antes.

Después de que el puerto `feedbackFast` genere una solicitud de reenvío, la instrucción correspondiente no seguirá fluyendo en la secuencia. En otras palabras, el puerto `feedbackSlow` no generará retroalimentación para esta instrucción.

La canalización de la dirección de almacenamiento (sta) tiene solo un puerto de retroalimentación. En la etapa de almacenamiento 1, la canalización de almacenamiento informa a la estación de reserva si es necesario volver a emitir la instrucción.

Además de la información sobre si se debe reenviar el comando, el puerto de comentarios de reenvío también incluye la siguiente información:

* Utilice el índice de la estación de reserva (rsIdx) para indexar la ubicación de la instrucción que se volverá a emitir en la estación de reserva.
* Utilice el campo sourceType para distinguir diferentes motivos de reenvío
* Proporciona una interfaz para la retroalimentación de este sqIdx de la tienda cuando la carga encuentra que la dirección de la tienda anterior está lista pero los datos no están listos

!!! nota
 Este mecanismo puede cambiar en la próxima versión del diseño.

## Almacenar para cargar hacia adelante

Store To Load Forward (STLF) significa que antes de que los datos de una instrucción de almacenamiento se escriban en la caché de datos, las instrucciones de carga posteriores que acceden a la misma dirección obtienen esta instrucción de almacenamiento de la cola de acceso a la memoria y del búfer en el núcleo. Operaciones sobre datos.

La operación de avance desde el almacenamiento hasta la carga se asigna a la secuencia de tres etapas. Durante la operación de avance, la lógica de avance verifica en paralelo si los datos requeridos por la carga actual existen en el búfer de almacenamiento y la cola de almacenamiento comprometidos. Si es así, los datos se fusiona en En el resultado de esta carga.

### Reenvío de dirección virtual

Por cuestiones de tiempo, la arquitectura de Southlake utiliza un mecanismo de verificación de dirección real y un pase de reenvío de dirección virtual. Este mecanismo optimiza el rendimiento de tiempo de reenvío de la tienda a la carga.

<!-- !!! todo -->
<!-- Idea básica, diagrama -->

**Ruta de datos para el reenvío de direcciones virtuales:** La etapa 0 de la [canalización de carga](../memory/fu/load_pipeline.md) genera una máscara para el reenvío de datos en función del sqIdx de la instrucción. En la etapa 1, la dirección virtual y la máscara se envían a la [cola de almacenamiento](../memory/lsq/store_queue.md#store-to-load-forward-query) y al [búfer de almacenamiento confirmado](../memory/lsq/ En la etapa 2 del pipeline de carga, la cola de almacenamiento y el buffer de almacenamiento confirmado generan resultados de consultas de reenvío, que se fusionan con los resultados leídos desde dcache.

**Ruta de control para reenviar direcciones virtuales:** En la etapa 0 de la secuencia de carga, la dirección virtual de la instrucción se envía a la TLB para iniciar la conversión de dirección virtual a real. En la etapa 1 de la secuencia de carga, la TLB devuelve la dirección física de la instrucción. La dirección física y la máscara se envían a la [cola de almacenamiento](../memory/lsq/store_queue.md#store-to-load-forward-query) y al [búfer de almacenamiento confirmado] ](../memory/lsq/committed_store_buffer.md#store -to-load-forward-query) realiza una consulta de reenvío (solo coincidencia de direcciones). En la etapa de carga 2, el resultado de la coincidencia de la dirección virtual y el resultado de la coincidencia de Se comparará la dirección real. Si las dos son diferentes, significa que se reenvía la dirección virtual. Se produjo un error durante el reenvío. [Después de verificar si hay un error, active una reversión y actualice el búfer de almacenamiento confirmado](../fu /load_pipeline.md#forward-failure). Esta operación elimina la dirección virtual que causó el error de la cola de almacenamiento y del búfer de almacenamiento confirmado. búfer de almacenamiento Excluido de.

!!! información
 En cambio, el proceso de reenvío de direcciones reales en la arquitectura de Yanqi Lake es el siguiente: la etapa 0 de la canalización de carga genera una máscara para el reenvío de datos en función del sqIdx de la instrucción. En la etapa 1 de la canalización de carga, la TLB retroalimenta La dirección física. Esta máscara de dirección física se envía a la cola de almacenamiento y al búfer de almacenamiento comprometido para la consulta de reenvío. En la etapa 2 de la canalización de carga, la cola de almacenamiento y el búfer de almacenamiento comprometido generan resultados de consulta de reenvío, que se fusionan con los resultados leídos. Desde el dcache. Ruta de control y datos. Todos siguen este proceso.

### Guardando los resultados del avance

Si DCache falla, se conserva el resultado de reenvío. El resultado de reenvío (máscara y datos) se escribirá en la cola de carga. Cuando DCache rellene el resultado más tarde, la cola de carga será responsable de fusionar los datos rellenados con el resultado de reenvío para Finalmente generar una cola de carga completa. resultado.

### Optimización del rendimiento relacionado con el avance

Cuando el dcache falla pero el pase hacia adelante se ejecuta por completo, el resultado de esta instrucción se puede volver a escribir directamente sin esperar a que el dcache devuelva los datos. Sin embargo, debido a consideraciones de tiempo (no hay tiempo para marcar esta instrucción como un éxito) estado), la arquitectura Nanhu entregará esta situación a la cola de carga. Dichas instrucciones establecerán directamente el indicador `datavalid` (que indica que los datos de carga son válidos) al actualizar la cola de carga. Como resultado, la cola de carga se actualizará inmediatamente. Tenga en cuenta que estas instrucciones no necesitan esperar el resultado de la recarga de dcache. Estas instrucciones se pueden seleccionar y volver a escribir directamente.

## Violación de carga de la tienda

Esta sección describe la detección y recuperación de violaciones de carga de almacenamiento. La verificación de violación de carga comienza cuando una instrucción de almacenamiento llega a la etapa 1. Si se encuentra una violación de carga durante la verificación, el almacenamiento que activó la violación de carga no se marca como violación de carga en El ROB. Estado *Comprometerse*. Al mismo tiempo, la operación de reversión se activará inmediatamente, sin esperar a que se confirme la instrucción de almacenamiento que activó la violación de carga. La sección de cola de carga describe el proceso detallado de [verificación y redirección] (../memoria/lsq/ carga_cola.md#almacenar---cargar-%E8%BF%9D%E4%BE%8B%E6%A3%80%E6%9F%A5%E7%9B%B8%E5 %85%B3%E6%9C %BA%E5%88%B6).

## Violación de carga de carga

Consulte [detección y recuperación de violaciones de carga de carga](../memory/lsq/load_queue.md#load---load-%E8%BF%9D%E4%BE%8B%E6%A3%80%E6%9F %E5%85%B3%E6%9C%BA%E5%88%B6)

## La carga vuelve a escribir en la contención del puerto

La arquitectura de Southlake proporciona dos puertos de escritura diferida de carga. Este puerto es responsable de escribir el resultado de la carga en la estación de reserva, la pila de registros y notificar al ROB que la instrucción ha completado su ejecución. Tanto la etapa 2 de la canalización de carga como La cola de carga puede usar este puerto para volver a escribir. Como resultado, ambos competirán por el derecho a usar este puerto.

Normalmente, las instrucciones de carga en la tubería tienen mayor prioridad.
