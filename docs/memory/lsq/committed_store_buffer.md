# Buffer de almacenamiento comprometido

Este capítulo presenta el diseño del búfer de almacenamiento comprometido (sbuffer) de la arquitectura Nanhu del procesador Xiangshan.

El búfer de almacenamiento comprometido de la arquitectura Southlake puede almacenar hasta 16 elementos de datos organizados en líneas de caché. El búfer de almacenamiento comprometido puede almacenar como máximo:

* Recibir 2 instrucciones de tienda escritas por la cola de la tienda
* Escribe una línea de caché en la caché de datos

El búfer de almacenamiento comprometido fusiona las solicitudes de escritura en el almacenamiento en la granularidad de la línea de caché. Cuando la capacidad excede el umbral establecido, el búfer de almacenamiento comprometido realizará una operación de intercambio, utilizará PLRU para seleccionar la línea de caché que se escribirá en la caché de datos y la escribirá en los datos. Caché. El umbral de intercambio automático para el búfer de almacenamiento comprometido se puede configurar utilizando el CSR `smblockctl`.

<!-- TODO: Imagen -->

## El contenido de cada elemento en el búfer

* Bits de estado
*Dirección virtual y real
* datos
* Máscara de validez de datos

Bits de estado | Descripción
-|-
state_valid|Este artículo es válido
state_inflight|Se ha enviado una solicitud de escritura a dcache y el resultado de la escritura está esperando a dcache.
w_timeout|esperar el tiempo de espera, esperar a que se agote el tiempo de espera del temporizador de reenvío para volver a enviar la solicitud de escritura a dcache
w_sameblock_inflight|con sameblock inflight, hay una entrada de sbuffer de la misma línea de caché que se escribe en el caché de datos. Las instrucciones en este estado no se seleccionarán para escribirse en el caché de datos para mantener el orden de las operaciones de escritura de sbuffer en el caché de datos. Consulte [Escritura entradas al dcache [seleccionar](#Seleccione el elemento a escribir en -dcache)

## mecanismo de escritura sq en sbuffer

Debido al tiempo, se necesitan dos tiempos para que sq escriba datos en el búfer.

* Primer disparo: leer datos y direcciones de sq y almacenarlos temporalmente en `DatamoduleResultBuffer`, ver [cola de almacenamiento](../lsq/store_queue.md).
* Segundo tiempo: utilice la dirección almacenada temporalmente en el tiempo anterior para compararla con la dirección/estado existente en el búfer y determinar si se puede escribir. Si se puede, realice la operación de escritura.

Las comprobaciones de dirección y estado realizadas en el segundo tiempo producen los siguientes resultados:

Verificar resultados | Acciones de seguimiento
-|-
No hay coincidencia en la dirección, la línea de caché no existe en el búfer | Asignar nueva entrada y escribir
La dirección coincide y no se ha iniciado ninguna solicitud de escritura en dcache para este elemento | Fusionar los datos escritos esta vez en el elemento existente en el búfer
La dirección tiene una coincidencia y el elemento está escribiendo una solicitud de dcache | solicitud de escritura de bloqueo
sbuffer sin elementos vacíos | bloqueando solicitudes de escritura

Si la escritura o la fusión se ejecutan correctamente, se actualizarán los datos y la máscara del elemento correspondiente. La dirección y la información de control también se escribirán en el búfer durante la primera escritura. Por cuestiones de tiempo, cuando no hay ningún elemento vacío en el sbuffer, incluso si también se producen la misma fusión y escritura de datos de Cacheline.

## Mecanismo de escritura de sbuffer en dcache

sbuffer escribe en dcache en dos pasos:

* Primer disparo: seleccione el elemento que se escribirá en dcache, lea los datos y el elemento correspondiente se marca como escrito en dcache
* Segundo disparo: la solicitud de escritura se envía a dcache

Después de que sbuffer envía una solicitud de escritura a dcache, esperará el resultado de escritura devuelto por dcache. Es posible que dcache no acepte la solicitud de escritura (error y MSHR está lleno, consulte la sección de la tubería principal de dcache) y la instrucción de almacenamiento en este momento iniciará la Reenviar el temporizador en el búfer de almacenamiento temporal y esperar. Cuando el temporizador de retransmisión llega a 0, el búfer de almacenamiento temporal intentará escribir este elemento en dcache nuevamente. Solo después de que dcache informe que la escritura de un elemento se realizó correctamente, el búfer de almacenamiento temporal marcará este elemento como no válido.

Hay dos tipos de interfaces de retroalimentación de dcache a sbuffer: interfaz de acierto e interfaz de reenvío. La interfaz de acierto puede provenir de MainPipe (acierto) o RefillPipe (error, pero la escritura se completa después de ser asignada a la entrada MissQueue). La interfaz de reenvío también Proviene de MainPipe (perdido y no asignado a la entrada MissQueue). Consulte [dcache mainpipe](../dcache/main_pipe.md) para obtener más detalles.

Durante la escritura de sbuffer en dcache, el proceso de cambio de estado de un elemento en sbuffer es el siguiente:

1. Válido, no se puede escribir en dcache
1. Válido, escribiendo en dcache
1. No se pudo escribir en dcache, está esperando a ser reenviado o no es válido

!!! nota
 El diseño del sbuffer permite que los datos de la misma línea de caché se escriban en el sbuffer mientras se escribe una entrada del sbuffer en el caché de datos. Los nuevos datos ocuparán dos entradas diferentes en el sbuffer que los datos que se escriben en el sbuffer. .

### Selección de elementos para escribir en dcache

Las entradas de sbuffer que cumplan las siguientes condiciones serán escritas en dcache por sbuffer, con las condiciones enumeradas en orden descendente de prioridad:

1. La solicitud de dcache se reenvía y alcanza el tiempo de espera.
1. actualización del búfer
1. El tiempo que un elemento ha estado en el búfer alcanza un umbral y se agota el tiempo de espera.
1. El búfer está casi lleno y algunos elementos están intercambiados.

La selección de elementos intercambiados utiliza el algoritmo PLRU. Cuando se escribe un almacenamiento en el búfer, o cuando se fusiona un almacenamiento con un elemento en el búfer, se actualiza el PLRU. Cuando el indicador `w_sameblock_inflight` de un elemento está alto (lo que significa que se utiliza la misma línea de caché), se actualiza el PLRU. Cuando se escribe otra entrada de sbuffer en el caché de datos, esta entrada no se seleccionará como el objeto que se escribirá en el caché de datos.

### Limpiar el búfer

Al limpiar activamente el búfer, este escribirá todas las instrucciones confirmadas en la cola de almacenamiento en el caché de datos. El búfer también se actualizará para admitir el mecanismo de reenvío de direcciones virtuales. Cuando el búfer detecta que el resultado del reenvío de direcciones virtuales tiene un conflicto (ver [Soporte de reenvío](./committed_store_buffer.md#store-to-load-forward-query)), el búfer también se actualizará automáticamente.
<!-- La cantidad de ciclos que activan el intercambio automático no admite la configuración manual en este momento. -->

## Consulta de avance de la tienda a la carga

Cuando la instrucción de carga realiza una consulta de almacenamiento a carga hacia adelante, el búfer también proporciona a la instrucción de carga los datos del almacenamiento que se han enviado pero no escrito en el caché de datos de dominio público. En el [mecanismo de avance](../mechanism.md actual #store-to- En el modo de reenvío de carga, el mecanismo de reenvío desde el búfer adopta el reenvío de direcciones virtuales y la verificación de direcciones reales.

### Generar resultados de pases hacia adelante

El sbuffer reenviará los datos generados por la dirección virtual en el próximo ciclo cuando llegue la solicitud de consulta de reenvío, lo cual es consistente con la cola de almacenamiento.

### Comprobación de corrección de reenvío

Para garantizar que el resultado del pase hacia adelante generado mediante el uso de la dirección virtual sea el resultado correcto, el búfer de almacenamiento realizará verificaciones relacionadas con el pase hacia adelante de la dirección virtual:

<!-- * **Verificar antes de la entrega.** -->

Una vez que los resultados coincidentes de las direcciones virtuales y reales son diferentes durante la entrega hacia adelante, se activa el búfer para que se actualice a la fuerza. Todos los elementos se escriben en el caché de datos para evitar que el búfer continúe produciendo resultados de entrega hacia adelante incorrectos. Al mismo tiempo El mensaje de error de entrega de reenvío se envía de vuelta a la tubería de carga. La tubería realiza el [manejo de errores de reenvío](../fu/load_pipeline.md#forward-failure).

<!-- * **Comprobación en tiempo de escritura.** La operación de escritura en el búfer de almacenamiento intentará fusionar la operación de escritura en la misma línea de caché en un elemento del búfer de acuerdo con la dirección real. Si se encuentra que la dirección real Es lo mismo pero la dirección virtual es diferente. Activa la actualización forzada del búfer. El elemento correspondiente en el búfer se actualizará a la nueva dirección virtual. -->

<!-- El mecanismo de verificación en tiempo de escritura se puede considerar cancelado y el búfer se actualiza de manera diferida solo cuando se encuentra un problema en la pasada anterior. -->
