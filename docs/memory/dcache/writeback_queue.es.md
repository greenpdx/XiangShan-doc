# Cola de escritura diferida

El DCache de la arquitectura NanHu utiliza una cola de escritura diferida de 18 elementos, que es responsable de liberar bloques de reemplazo (Release) a la caché L2 a través del canal C del TL-C, o responder a solicitudes de sonda (ProbeAck), y Admite la comunicación entre Release y ProbeAck. Se fusionan entre sí para reducir la cantidad de solicitudes y optimizar los tiempos.

## Entrada de cola de escritura diferida

Por razones de tiempo, cuando wbq está lleno, se rechazarán las nuevas solicitudes independientemente de si se pueden fusionar; cuando wbq no está lleno, se aceptarán todas las solicitudes y se asignará un elemento vacío para la nueva solicitud o la nueva solicitud se fusionará con un elemento existente. En la entrada de escritura diferida, verá en la sección [Mantenimiento de estado](#writeback-queue-State Maintenance) que la entrada de escritura diferida se puede fusionar con una nueva solicitud de versión o ProbeAck en cualquier momento. Por lo tanto, la arquitectura NanHu determina si la cola de escritura diferida se puede unir. Para unirse a una cola, solo necesita ver si hay elementos vacíos en la cola, lo que acorta en gran medida el retraso lógico de unirse a la cola.

## Mantenimiento del estado de la cola de escritura diferida

Cada entrada de escritura diferida tiene los 4 estados siguientes:

Estado|Descripción
-|-
`s_invalid`|La entrada de escritura diferida es una entrada vacía
`s_sleep`|Prepárese para enviar una solicitud de liberación, pero duerma temporalmente y espere una solicitud de recarga para despertar
`s_release_req` | ​​Envío de una solicitud de Release o ProbeAck
`s_release_resp`|Esperando la solicitud ReleaseAck

!!! información
 **Nota sobre el estado `s_sleep`**: Por razones de rendimiento, no queremos que el bloque de reemplazo se invalide demasiado pronto, para evitar que el núcleo acceda nuevamente al bloque de reemplazo durante el tiempo de acceso a L2 / L3 hacia abajo. lo que genera un efecto ping-pong, generando nuevas solicitudes de falla innecesarias. Por lo tanto, la solicitud de reemplazo ingresa a la tubería principal y a la cola de escritura de forma sucesiva, no para invalidar realmente el bloque de reemplazo, sino para leer primero los datos del bloque de reemplazo y poner temporalmente en la cola de reescritura. Durante el período de suspensión de la solicitud de reemplazo, otras solicitudes aún pueden acceder al bloque de reemplazo en DCache normalmente, siempre que la escritura del bloque de reemplazo esté sincronizada con la cola de reescritura. Cuando se completa el reabastecimiento Cuando se toma un bloque, se puede reactivar la escritura. Se devuelve el bloque inactivo de la cola de escritura y la cola de escritura comienza a liberar el bloque de reemplazo hacia abajo. Al mismo tiempo, la cola de faltas solicita a la tubería de recarga que complete el relleno. El bloque de reemplazo se sobrescribirá durante el relleno.

Sin considerar la fusión de solicitudes, el proceso de procesamiento de entrada de escritura inversa ProbeAck y Release es el siguiente:

1. ProbeAck: `s_invalid` -> `s_release_req` -> `s_invalid`

2. Liberación: `s_invalid` -> `s_sleep` -> `s_release_req` -> `s_release_resp` -> `s_invalid`

Si hay solicitudes que se pueden fusionar, el proceso es un poco más complicado:

3. Fusión de ProbeAck de Release: En cualquier etapa del flujo de procesamiento de Release anterior, es posible recibir ProbeAck de la misma dirección: si está en la etapa `s_sleep`, la solicitud de Release se convierte directamente en una solicitud ProbeAck; si está en la etapa `s_sleep`, la solicitud de Release se convierte directamente en una solicitud ProbeAck; está en la etapa `s_release_req`, si la solicitud de lanzamiento no ha completado el protocolo de enlace, también puede enviar directamente el lanzamiento
Si la solicitud de liberación ha completado al menos un protocolo de enlace en la fase `s_release_req`, entonces la solicitud ProbeAck es demasiado tarde. En este caso, se establece el bit `release_later` y se registra la información relacionada con ProbeAck. Una vez que se completa todo el procesamiento, ProbeAck está procesado.

4. Versión de fusión de ProbeAck: dado que esta parte es muy detallada, no la explicaré con más detalle. Para obtener contenido específico, consulte el código de `src/main/scala/xiangshan/cache/dcache/mainpipe/WritebackQueue.scala `. La idea principal es fusionar siempre que sea posible, hacer la menor cantidad de lanzamientos posible y, si un lanzamiento llega demasiado tarde, configurar el bit `release_later` para procesarlo más tarde.

## Bloquear solicitudes de cola perdida

Las restricciones del manual de TileLink sobre transacciones concurrentes requieren que si el maestro tiene una concesión pendiente (es decir, aún no ha enviado un GrantAck), no puede enviar una liberación con la misma dirección. Por lo tanto, todas las solicitudes de falta se enviarán a la cola de faltas Si encuentran que la dirección es la misma que la de un elemento en la cola de escritura diferida, entonces se bloqueará la solicitud faltante.
