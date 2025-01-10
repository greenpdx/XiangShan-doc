# Cola de carga

Este capítulo presenta el diseño de la cola de carga de la arquitectura Nanhu del procesador Xiangshan.

La cola de carga de la arquitectura Southlake es una cola circular de 80 elementos. Cada ciclo tiene como máximo:

* Recibir 4 instrucciones del despacho
* Recibir los resultados de 2 instrucciones del pipeline de carga para actualizar su estado interno
* Recibir resultados de recarga fallida de dcache y actualizar el estado de todas las instrucciones en la cola de carga que están esperando esta recarga
* Vuelva a escribir las dos instrucciones de carga que faltaban. Estas instrucciones ya obtuvieron los datos de recarga y competirán con el canal de acceso a la memoria normal por dos puertos de escritura diferida.

Cuando se vuelve a llenar el caché de datos de primer nivel, todos los datos de una línea de caché completa se devolverán a la cola de carga, y todas las instrucciones que esperan los datos de esta línea de caché en la cola de carga obtendrán los datos.

## Cargar el contenido de la cola de cada elemento

Cada entrada de la cola de carga contiene la siguiente información:

* Dirección física
* Dirección virtual (depuración)
* Recargar elementos de datos
* Bits de estado
* Bits de estado utilizados por el disparador

Bits de estado | Descripción
-|-
asignado|El artículo ha sido asignado por envío
El comando datavalid|load ha obtenido los datos requeridos
El resultado de la instrucción writebacked|load se ha vuelto a escribir en el archivo de registro y se ha notificado a ROB, RS
instrucción miss|load omitió dcache, esperando recarga de dcache
pendiente|La instrucción de carga accede al espacio de direcciones mmio y se pospone la ejecución. Esperando a que la instrucción se convierta en la última instrucción en el ROB
liberado|La línea de caché a la que accedió la instrucción de carga ha sido liberada por dcache (liberación)
error|load Se detectó un error durante la ejecución de la instrucción

## Cargar cola Poner en cola

La instrucción de carga que ingresa a la cola de carga en realidad se completa en dos pasos: la asignación anticipada de `enqPtr` y la escritura real en la cola de carga.

El motivo de la asignación temprana es que el módulo de despacho está demasiado lejos de MemBlock y el envío del `enqPtr` generado en la cola de carga al despacho como `lqIdx` enfrentará una demora prolongada. La arquitectura de Southlake mantiene la lógica de asignación temprana de `enqPtr` cerca del despacho. Es responsabilidad de la lógica de despacho temprana proporcionar el `lqIdx` de la instrucción.

<!-- ### Asignación anticipada de enqPtr

!!! hacer
 Consulte la sección de despacho, [asignación anticipada](../../backend/dispatch.md) de `lqIdx` / `sqIdx`. -->

### Cola de carga Escritura real

Cuando se escribe realmente la cola de carga, el `enqPtr` de la cola de carga se actualiza de acuerdo con la cantidad de instrucciones escritas en la cola de carga. Por razones de tiempo, la cola de carga solo se actualizará cuando la cantidad de elementos vacíos en la cola de carga sea mayor. La cola de carga es >= el número de instrucciones enq. ` acepta las instrucciones enviadas.

## Actualizar cola de carga

Durante la ejecución de una instrucción en la cadena de carga, el estado de la cola de carga se puede actualizar en varias etapas.

### Etapa de carga 2

En esta etapa, dcache y la recursión hacia adelante devuelven los resultados y escriben lo siguiente en la cola de carga:

* Señales de control clave
* Dirección física
* Resultados adelantados
* resultado de la comprobación del disparador

Los indicadores de estado que se actualizarán en este momento incluyen: `datavalid`, `writebacked`, `miss`, `pending`, `released`

Las instrucciones MMIO actualizan la cola de carga de forma diferente a las instrucciones normales.

### Etapa de carga 3

Durante esta fase, algunas de las comprobaciones que no se completaron en el ciclo anterior devuelven resultados, y la cola de carga utilizará estos resultados para actualizar su estado. Por ejemplo, una vez que se produce el evento `dcacheRequireReplay` (dcache solicita que se vuelvan a enviar instrucciones desde el estación de reserva) se activa, los indicadores ` miss y datavalid se actualizan a falso. Esto indica que la instrucción se volverá a emitir desde la estación de reserva en lugar de esperar en la cola de carga a que se rellene para despertarla. Esta operación es coherente con la Retroalimentación de la estación de reserva en la etapa 3 del proceso de instrucciones de carga. Las operaciones se realizan de manera sincrónica.

!!! información
 El procesamiento de [error de carga](../fu/load_pipeline.md#load-miss) en la canalización de carga también involucra información relevante sobre las actualizaciones de la cola de carga.

## Cargar recarga

Si se asigna correctamente una instrucción de carga a dcache MSHR, escuchará el resultado de la recarga de dcache en la cola de carga. Una recarga pasará los datos a todos los elementos de la cola de carga que esperan esta línea de caché. El estado de los datos de estos elementos se marca como válido, que luego se puede volver a escribir. Si la instrucción se ha reenviado de la tienda a la carga antes, la cola de carga es responsable de fusionar los resultados de reenvío al rellenar, consulte [Reenvío de la tienda a la carga](../mechanism.md#store - El siguiente diagrama muestra los cambios en los elementos de la cola de carga antes y después de una recarga de dcache.

<!-- !!! todo
 Se actualizó la descripción del diagrama -->

![antes de rellenar](../../figs/memblock/antes-de-rellenar.png)

![después del rellenado](../../figs/memblock/after-refill.png)

Una vez que la cola de carga obtiene los datos de la recarga de caché de datos, puede comenzar a escribir las instrucciones de carga que faltan en la cola de carga. La cola de carga proporciona dos puertos para este tipo de operación de escritura diferida de instrucciones. En cada ciclo, la cola de carga selecciona la instrucción más antigua de las columnas pares e impares que ha completado la recarga pero que aún no se ha vuelto a escribir, y la vuelve a escribir a través del puerto de escritura diferida en el siguiente ciclo (por consideraciones de tiempo, la selección de instrucciones de escritura diferida y la escritura real) -back se ejecuta dos ciclos antes y después. La cola de carga competirá con las instrucciones de carga ejecutadas normalmente en la canalización de carga por el puerto de escritura diferida. Cuando las instrucciones en la canalización de carga intentan escribir diferidamente, la solicitud de escritura diferida de La cola de carga está bloqueada.

## Finalización de la instrucción de carga

La finalización de la instrucción de carga se refiere a la operación de escribir el resultado de la carga de nuevo en rob y rf. La selección de la carga para escribir de nuevo en ROB y RF se divide en partes pares e impares, y la instrucción más antigua se selecciona primero. Se seleccionan dos instrucciones para escribir como máximo por ciclo. Hay dos instrucciones para seleccionar la escritura diferida en la cola de carga:

* Instrucciones de carga de mmio completadas
* Anteriormente se omitió, ahora se utilizó la instrucción de carga para obtener el resultado de los datos rellenados desde dcache

!!!información
 Las cargas normales que llegan a dcache se volverán a escribir directamente desde la canalización. Consulte la sección [canalización de carga](../fu/load_pipeline.md#stage-2).

La operación de escritura diferida real se produce en el siguiente latido de la selección de escritura diferida. La cola de carga leerá la información de instrucción correspondiente según el resultado de la selección, completará el recorte del resultado según los requisitos de la instrucción y, finalmente, utilizará la operación de escritura diferida de carga. puerto de retorno para escribir el resultado. La carga se escribe correctamente. Una vez que se completa la escritura, el indicador `writebacked` se actualizará a falso.

!!! nota
 Tenga en cuenta la diferencia entre las instrucciones de carga que se vuelven a escribir en la cola de carga y las instrucciones de carga que se vuelven a escribir desde la cola de carga a rob y rf. Se trata de operaciones diferentes.

## Mecanismo relacionado con el envío de carga

Después de enviar el comando, rob genera una señal `lcommit` según la cantidad de comandos de carga enviados, notificando a la cola de carga que esta cantidad de comandos de carga se han enviado correctamente.

Dado que la cola de carga está lejos del ROB, la cola de carga en realidad usa `lcommit` para actualizar el estado interno dos tiempos después de que la instrucción se confirma en el ROB. La cola de carga actualizará el indicador `allocated` de la instrucción confirmada a falso para indicar su finalización. Al mismo tiempo, actualice el puntero de cola de la cola `deqPtr` según la cantidad de cargas enviadas (`lcommit`).

## redirigir

En esta sección se presenta el mecanismo relacionado con la redirección de comandos en la cola de carga. Una vez que la redirección llega a la cola de carga, el estado de la cola de carga se actualizará en 2 segundos:

* Ciclo 1: busca todas las instrucciones en la ruta incorrecta según robIdx. La asignación de las instrucciones eliminadas se establece en falso.
* Ciclo2: Según el resultado de la búsqueda anterior, cuente cuántas instrucciones deben cancelarse y actualice enqPtr

En el diseño actual, después de que la instrucción de salto activa la redirección, aún puede haber instrucciones de carga válidas en la cola de despacho que deben ingresar a la cola de carga. Cuando cycle2 realiza la actualización enqPtr, ¿se actualizan las instrucciones que ingresan a la cola de carga durante la redirección? Las cancelaciones se contabilizan aparte.

<!-- TODO: Procesamiento de la instrucción del ciclo 2 enq -->

<!-- ### Mantenimiento del puntero de cola -->

## tienda - mecanismo de verificación de violación de carga

Mientras la operación de almacenar dirección escribe la dirección en la cola de almacenamiento (etapa 1 de la canalización de almacenar dirección), también busca en la cola de carga instrucciones de carga con la misma dirección física pero en el orden posterior al almacenamiento. Si se han ejecutado estas instrucciones de carga y si se genera un resultado incorrecto (es decir, se activa una violación de carga de almacenamiento), la cola de carga emitirá una solicitud de redirección: el procesador comenzará desde esta instrucción de carga y volverá a ejecutar las instrucciones subsiguientes comenzando desde la búsqueda de instrucciones.

Para dos canales de almacenamiento, cada instrucción debe verificar si las instrucciones de carga en tres ubicaciones están en violación (canal de carga etapa 1/etapa 2, cola de carga), y se verificará un total de 6 posibles violaciones. La lógica de verificación de violaciones debe Se selecciona la más antigua de las posibles violaciones y se genera una solicitud de redirección. Para ello, para cada instrucción de almacenamiento, su verificación de violación de carga de almacenamiento se divide en tres ciclos para su ejecución, y las operaciones realizadas en cada ciclo se como sigue :

* Ciclo 0: la dirección de la tienda actualiza la cola de la tienda
 * La tubería de la tienda envía la dirección física a la cola de carga y genera un vector coincidente basado en la coincidencia de la dirección.
 * Calcule el rango de verificación en función del lqIdx proporcionado con la instrucción de almacenamiento
 * Verificar si las instrucciones en la etapa 1 / etapa 2 del pipeline de carga utilizan el resultado de este almacenamiento
* Ciclo 1: Generación de redireccionamiento
 * Comprobación completa de violaciones en la cola de carga
 * Si se produjo una infracción en la etapa 1/etapa 2 del pipeline de carga anterior, seleccione la más antigua
* Ciclo 2: Generación de redireccionamiento
 * Seleccione la infracción más antigua entre todas las infracciones
 * Genera una solicitud de redireccionamiento saliente

```
 etapa 0: lq l1 l2 l1 l2 lq
 | | | | | | (coincidencia de paddr)
 Etapa 1: lq l1 l2 l1 l2 lq
 | | | | | |
 | |------------| |
 | |
