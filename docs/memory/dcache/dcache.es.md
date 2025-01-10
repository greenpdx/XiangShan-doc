# Caché de datos de nivel 1

DCache está estrechamente acoplado con el canal de acceso a la memoria en la arquitectura Nanhu e interactúa directamente con la caché L2 a través del protocolo de bus TileLink. El DCache predeterminado en la arquitectura Nanhu tiene una capacidad total de 128 KB, una estructura asociativa de 8 vías, reemplazo pseudo LRU y SECDED. Verificar. Funciona con caché l2 para manejar el [problema de alias de caché](../../huancun/cache_alias.md) causado por una capacidad de 128 KB. La arquitectura Nanhu dcache admite [operaciones de caché](./csr_cache_op.md) personalizadas.

Los módulos internos de DCache incluyen:

Nombre del módulo | Descripción
-|-
[Load PipeLine](./load_pipeline.md) (tubería de carga) | Estrechamente acoplada con la tubería de acceso de carga, lectura de 3 pasos
[Main Pipeline](./main_pipe.md) (Main pipeline) | Responsable de la ejecución de operaciones de almacenamiento, sondeo, reemplazo y atómicas
Canal de recarga | Responsable de escribir los datos de recarga de L2 en dcache
[Unidad atómica](../fu/atom.md#dcache-Compatibilidad con instrucciones atómicas) (AtomicsReplayEntry) | Programación de solicitudes atómicas
[Miss Queue](./miss_queue.md) (MSHR, 16 elementos) |Solicita bloques faltantes de L2, cada elemento controla el flujo de la solicitud faltante en DCache a través de una máquina de estados
[Cola de sondeo](./probe_queue.md) (8 elementos)|Recibir solicitudes de coherencia de caché L2
[Cola de escritura diferida](./writeback_queue.md) (18 elementos) | Responsable de escribir el bloque de reemplazo nuevamente en la caché L2 o de responder a la solicitud de sonda

[Committed Store Buffer](../lsq/committed_store_buffer.md) también servirá como un búfer de reenvío para solicitudes de escritura mientras se envían solicitudes de escritura a DCache. El diagrama de estructura general de DCache y Committed Store Buffer es el siguiente:

![dcache](../../figs/memblock/dcache.png)

## Interfaz

El DCache en la arquitectura de Yanqi Lake utiliza el protocolo de bus TileLink para caché L2. Las solicitudes de consistencia involucradas en TileLink se dividen principalmente en tres categorías:

* Adquirir Obtener permiso
* Permiso de liberación pasiva de la sonda
* Liberar Liberar activamente los permisos

<!-- !!! todo
 Figura a actualizar -->

<!-- ![enlace de mosaico](../../figs/memblock/dcache-tilelink-interface.jpg) -->

## Flujo de procesamiento de solicitudes

### Flujo de procesamiento de solicitud de carga

* Después de tres niveles de pipeline, si impacta, será devuelto directamente, si falla, entrará en Miss Queue
* Miss Queue recupera datos de relleno y los reenvía a Load Queue
* Escribir datos de relleno en DCache en la canalización principal
* Si es necesario reemplazar un bloque, vuelva a escribir el bloque de reemplazo en la cola de escritura diferida.

### Flujo de procesamiento de solicitudes de tienda

La unidad de reproducción de tienda recibe solicitudes del búfer de tienda y accede a DCache en la tubería principal:

* Si DCache impacta
 * Escritura completa en DCache en el pipeline principal
* Si el DCache falla y la cola de solicitudes perdidas está llena (o se niega a aceptar la solicitud)
 * El Store Buffer volverá a enviar esta solicitud después de un período de tiempo.
* Si DCache falla y Miss Queue acepta correctamente la solicitud
 *La Miss Queue continúa realizando operaciones posteriores
 * Notificar al búfer de tienda una vez completado y actualizar dcache a través de la tubería de recarga
* Si hay un bloque reemplazado, escríbalo nuevamente en la cola de escritura diferida
 * Específicamente, un bloque reemplazado se escribe en la cola de escritura solo después de que el bloque que lo reemplazó llega a DCache.


### Flujo de procesamiento de solicitudes de Atomics

Consulte [Flujo de procesamiento de instrucciones atómicas](../fu/atom.md).

<!-- ### Flujo de procesamiento de solicitud de recarga

!!! hacer
 Para actualizar -->

<!-- ### Reemplazar el disparador y el flujo de procesamiento -->

### Flujo de procesamiento de solicitud de sonda

* Recibir solicitudes de sonda desde la caché L2
* Modificar los permisos de los bloques sondeados en la tubería principal
* Devuelve la respuesta y vuelve a escribir los datos sucios

<!-- ## Contención de recursos

!!! hacer
 Los componentes de dcache competirán por recursos como matrices de datos, matrices de etiquetas, MissQueue, Writeback Queue, etc. Esta sección describirá cómo dcache programa estos recursos. -->
