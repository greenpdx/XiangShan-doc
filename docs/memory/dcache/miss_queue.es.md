# Señorita Cola

En la arquitectura NanHu, la cola de fallas contiene 16 entradas de fallas. Cada entrada de falla es responsable de recibir solicitudes de carga, almacenamiento y atómicas de fallas, recuperar los datos que se deben rellenar de la caché L2 y devolver los datos de carga faltantes a la Cola de carga.

Tomando la carga como ejemplo, el flujo de procesamiento de solicitudes de error comunes en Miss Queue es el siguiente:

* Asignar una entrada de señorita vacía en la cola de señoritas y registrar la información relevante en la entrada de señoritas;
* Determinar si el bloque donde se encuentra `way_en` es válido y, de ser así, enviar una solicitud de reemplazo a la tubería principal;
* Envíe una solicitud de adquisición a L2 al mismo tiempo que envía una solicitud de reemplazo. Si se va a sobrescribir todo el bloque, envíe AcquirePerm (L2 guardará una operación de lectura de sram); de lo contrario, envíe AcquireBlock;
* Esperar a que L2 devuelva el permiso (Grant) o datos más permiso (GrantData);
* Si se produce un error de carga, los datos se deben enviar a la cola de carga después de recibir cada latido de GrantData;
* Devuelve GrantAck a L2 después de recibir el primer latido de Grant/GrantData;
* Después de recibir el último latido de Grant/GrantData y la solicitud de reemplazo se haya completado, envíe una solicitud de recarga al conducto de recarga y espere una respuesta para completar la recarga de datos;
* Liberar a la señorita Entrada.

El proceso de error de almacenamiento y error de carga es básicamente el mismo, la diferencia es que no es necesario reenviar los datos rellenados a la cola de carga. Además, después de que el error de almacenamiento finalmente complete el relleno, debe devolver una respuesta a la cola de carga. Buffer de tienda para indicar que el almacenamiento se ha completado.

El flujo de procesamiento de una instrucción atómica fallida en la cola de fallas es el siguiente:

* Asignar una entrada de señorita vacía en la cola de señoritas y registrar la información relevante en la entrada de señoritas;
* Enviar solicitud AcquireBlock a L2;
* Espere a que L2 devuelva GrantData;
* Devuelve GrantAck a L2 después de recibir el primer latido de GrantData;
* Después de recibir el último latido de GrantData, se envía una solicitud a Main Pipe y se completan el reemplazo y el relleno en Main Pipe al mismo tiempo. Después de la finalización, se devuelve una respuesta a Miss Entry;
* Liberar a la señorita Entrada.

El flujo de procesamiento de solicitudes de fallas anterior está controlado por un conjunto de registros de estado en Miss Entry, que se describirán en detalle en [Mantenimiento del estado de la cola de fallas](#miss-queue-status-maintenance).


## Mantenimiento del estado de la cola de señoritas

Miss Entry se controla mediante una serie de registros de estado para controlar qué operaciones deben completarse y el orden en que se ejecutan estas operaciones. Como se muestra en la figura siguiente, los registros `s_*` representan las solicitudes que deben programarse. y los registros `w_*` representan las respuestas que se deben esperar. Los registros se establecen inicialmente en `true.B`. Cuando se asigna una entrada fallida a una solicitud, se asignan los registros `s_*` y `w_*` correspondientes. Establecido en `false.B`. El primero indica que se produjo una Miss Entry. La solicitud saliente aún no se envió, lo que significa que hay que esperar una respuesta pero no se completó ningún protocolo de enlace.

![dcache-miss-entry.png](../../figs/memblock/dcache-miss-entry.png)

<!-- <div align="center">
<img src=../../figs/memblock/dcache-miss-entry.png ancho=60%>
<div> -->

Cada evento en Miss Entry se ejecuta en secuencia según ciertas dependencias. La figura anterior es un diagrama de flujo de DAG. La flecha indica que el próximo evento solo se puede ejecutar después de que el registro de estado anterior se configure en `true.B`.

Estado|Descripción
-|-
`s_acquire` | Envía AcquireBlock / AcquirePerm a L2. Si es necesario sobrescribir el bloque faltante con el bloque completo, solo se necesita AcquirePerm.
`w_grantfirst`|Primer latido de GrantData recibido
`w_grantlast`|El último latido de GrantData recibido
`s_grantack`|Indica que después de recibir datos L2, se devuelve una respuesta a L2. GrantAck se puede devolver cuando se recibe el primer latido de Grant
`s_mainpipe_req` | ​​Envía una solicitud atómica a Main Pipe para rellenar DCache
`w_mainpipe_resp`|Indica que después de enviar la solicitud atómica al Main Pipe y volver a llenarla en DCache, se recibe la respuesta del Main Pipe
`s_replace_req`|Se requiere reemplazo. Antes de ingresar a la Cola de fallas, la solicitud de carga/almacenamiento seleccionará una ruta de reemplazo de acuerdo con el algoritmo de reemplazo. Después de ingresar a la Cola de fallas, la solicitud se enviará directamente a la Tubería de reemplazo.
`w_replace_resp` | Completar el reemplazo
Las solicitudes de carga/almacenamiento `s_refill`|se deben enviar a la tubería de recarga para su posterior rellenado
`w_refill_resp`|Indica que la recarga está completa.


## Falta lógica de asignación de cola

La cola de solicitudes fallidas DCache de XiangShan admite un cierto grado de fusión de solicitudes, lo que mejora la eficiencia del procesamiento de solicitudes fallidas. En esta sección se presentará la estrategia de asignación y fusión de la entrada fallida en la arquitectura NanHu y en qué circunstancias se deben rechazar las nuevas solicitudes fallidas.

### Solicitar condiciones de fusión

Cuando las direcciones de bloque de la entrada faltante asignada (solicitud A) y la nueva solicitud faltante B son las mismas, se pueden fusionar en los dos casos siguientes:

1. La solicitud de adquisición de la solicitud A aún no ha tenido un protocolo de enlace, y A es una solicitud de carga, y B es una solicitud de carga o almacenamiento;
2. Se ha enviado la solicitud de adquisición de la solicitud A, pero no se ha recibido la concesión (datos), o se ha recibido la concesión (datos), pero no se ha reenviado a la cola de carga. A es una solicitud de carga o almacenamiento , y B es una solicitud de carga.

La primera condición se puede fusionar porque, mientras la adquisición no haya completado aún el protocolo de enlace, se pueden modificar los parámetros de la solicitud de adquisición. Cabe señalar que, después de fusionar el error de almacenamiento con la entrada de error para el error de carga, los datos de relleno Todavía necesita ser enviado a la cola de carga.

La segunda condición se puede fusionar porque mientras la entrada perdida de la dirección no haya enviado datos a la cola de carga, se puede fusionar una nueva solicitud de carga. Después de que la cola perdida obtenga los datos de relleno, activará todas las entradas en espera. Cargar colas de una vez. Se cargan los datos.

### Condiciones de rechazo de solicitud

Miss Queue rechazará nuevas solicitudes de falta en los dos casos siguientes:

1. La dirección de bloque de la nueva solicitud de falta es la misma que la solicitada en una entrada de falta, pero no cumple con la [condición de fusión de solicitud](#condición de fusión de solicitud);
2. El bloque solicitado por la nueva señorita tiene una dirección diferente del bloque solicitado en una entrada de señorita, pero está ubicado en la misma ranura en el DCache (es decir, ambos están en el mismo conjunto y manera).

Caso 1: Miss Entry ya no puede fusionar solicitudes de falta con la misma dirección por alguna razón, pero la canalización de carga y la canalización de almacenamiento (canalización principal) no pueden ser bloqueadas por Miss Queue, por lo que Miss Queue debe rechazar la solicitud de falta. /solicitud de almacenamiento Esperará un rato antes de volver a intentarlo.

Caso 2: En la versión NanHu, para escribir el bloque faltante de L2 a DCache inmediatamente, la solicitud de falta de carga/almacenamiento determinará la ruta de reemplazo antes de ingresar a la cola de falta, de modo que no haya necesidad de hacer otra comparación de etiquetas y Luego, decide reemplazar la ruta. Esto generará un nuevo problema: supongamos que dos solicitudes de carga en el mismo conjunto, pero con diferentes etiquetas, fallan en la tubería de carga una tras otra, pero las dos cargas deciden reemplazar la misma ruta. Una entrada fallida es asignados a cada uno de ellos, lo que eventualmente hace que el bloque que se rellene más tarde sobrescriba el bloque que se rellenó antes. Esto no solo generará problemas de rendimiento, sino que también hará que se pierdan los datos sucios si el bloque que se rellenó antes contiene datos sucios. Datos. Esto introduce el problema de la corrección. Por lo tanto, se debe establecer la segunda condición para garantizar que no haya dos direcciones de ranura idénticas en la cola de faltantes.

### Condición de asignación de elemento vacío de cola perdida

Cuando una nueva solicitud de error cumple con la [condición de fusión](#Condición de fusión de solicitud) o la [condición de rechazo](#Condición de rechazo de solicitud) mencionadas anteriormente, acepte o rechace la solicitud según corresponda. Las condiciones de fusión y rechazo aquí son completamente excluyentes entre sí. Sí, No habrá ningún conflicto entre ambos.

Por último, si no hay ninguna entrada perdida y desea fusionar o rechazar nuevas solicitudes perdidas:

* Si hay un elemento vacío en la cola de faltantes, asigne una nueva entrada de faltante;
* Si la cola de faltas está llena, se rechazarán nuevas solicitudes de faltas y la solicitud se reproducirá después de un cierto período de tiempo.


## Miss Queue activa el reemplazo

En la versión NanHu, en caso de error de carga o almacenamiento, la cola de errores determinará la ruta que se reemplazará cuando asigne una nueva entrada de error, de modo que se pueda rellenar inmediatamente después de recibir el bloque que se debe rellenar. Por este motivo, DCache necesita Para que se pueda reemplazar con antelación, se debe leer al menos el bloque de reemplazo antes de que se produzca el relleno. Por lo tanto, en esta versión, la entrada faltante se puede reemplazar inmediatamente después de su asignación, es decir, se envía una solicitud de reemplazo a la tubería principal.

Por razones de rendimiento, no queremos que el bloque de reemplazo se invalide demasiado pronto, para evitar que el núcleo acceda nuevamente al bloque de reemplazo durante el tiempo de acceso a L2/L3 hacia abajo, lo que genera un efecto ping-pong y genera nuevos errores innecesarios. solicitudes. Por lo tanto, el reemplazo mencionado aquí no invalida realmente el bloque de reemplazo, sino que lee primero los datos del bloque de reemplazo y lo coloca temporalmente en la cola de escritura diferida para que se suspenda. Durante el período de suspensión de la solicitud de reemplazo, otras solicitudes pueden Todavía se puede acceder a DCache con normalidad. Para reemplazar un bloque, simplemente sincronice la escritura del bloque de reemplazo con la cola de reescritura. Cuando se activa el bloque de relleno, se puede reactivar el bloque inactivo en la cola de reescritura y se puede iniciar la escritura. -La cola de retroceso comienza a liberar el bloque de reemplazo hacia abajo y, al mismo tiempo, la cola de fallas solicita a la tubería de recarga que complete el relleno, y el bloque de reemplazo se sobrescribirá durante el relleno.

## Recarga de cola de señorita

Véase [Tubería de recarga](./refill_pipe.md).
