# Documentación de la caché de instrucciones
<!-- Esta imagen necesita ser redibujada -->
![icache](../figs/frontend/ICache.png)

Este capítulo describe la implementación de la caché de instrucciones del procesador Xiangshan.

## Configuración de caché de instrucciones
| Nombre del parámetro | Descripción del parámetro |
| ------------ | ------------------------------------ |
| `nSet` | Número de conjuntos de caché de instrucciones, el valor predeterminado es 256 |
| `nWays` | La cantidad de formas en que cada conjunto en la caché de instrucciones puede conectarse. El valor predeterminado es 8. |
| `nTLBEntries` | El número de entradas en ITLB, el valor predeterminado es 40 entradas (32 páginas normales + 8 páginas grandes) |
| `tagECC` | Método de verificación Meta SRAM, la versión de Nanhu está configurada como verificación de paridad |
| `dataECC` | Método de verificación de SRAM de datos, la versión de Nanhu está configurada como verificación de paridad |
| `replacer` | Estrategia de reemplazo, el valor predeterminado es el reemplazo aleatorio, la versión de Nanhu está configurada como PLRU |
| `hasPrefetch` | Interruptor de precarga de instrucciones, el valor predeterminado es desactivado |
| `nPrefetchEntries` | La cantidad de entradas para la precarga de instrucciones y la cantidad máxima de precargas de líneas de caché que se pueden admitir al mismo tiempo. El valor predeterminado es 4 |


## Lógica de control
<!-- Diagrama lógico interno del módulo de lógica de control principal MainPipe de la caché de instrucciones: -->

El módulo de lógica de control principal MainPipe de la caché de instrucciones consta de una tubería de 3 etapas:

- En la fase `s0`, se envía una solicitud de búsqueda que contiene dos líneas de caché desde FTQ, que contiene una señal sobre si cada solicitud es válida (los paquetes de instrucciones que no cruzan líneas solo enviarán una solicitud de lectura para una línea de caché). Al mismo tiempo, MainPipe extrae la dirección de la solicitud como un índice de conjunto de caché y la envía a la [parte de almacenamiento](#imem) de la caché de instrucciones. Por otro lado, estas solicitudes se envían a ITLB para [traducción de la dirección de la instrucción](#itlb)
- En la fase `s1`, la SRAM de almacenamiento devuelve los metadatos de la línea de caché y los datos de un conjunto de caché con un total de N vías de caché. Al mismo tiempo, ITLB devuelve la dirección física correspondiente a la solicitud. A continuación, la lógica de control principal intercepta la dirección física y la compara con las etiquetas de caché de N vías, generando dos resultados: acierto de caché y error de caché. Además, la línea de caché que debe reemplazarse se seleccionará en función de la información de estado del algoritmo de reemplazo.
- En la fase `s2`, la solicitud de visita devuelve los datos directamente a la IFU. Cuando ocurre un error, es necesario pausar el proceso y enviar la solicitud a la unidad de procesamiento faltante (MissUnit). Espere hasta que MissUnit se complete y devuelva los datos antes de devolver los datos a IFU. En esta etapa, la dirección física traducida por ITLB se envía al módulo PMP para la consulta de permisos de acceso. Si el permiso es incorrecto, se activará una excepción de acceso a instrucciones (Instruction Access Fault).

<h2 id=itlb> Traducción de direcciones de instrucciones </h2>

Dado que la caché de instrucciones utiliza el método de caché VIPT (etiqueta física de índice virtual), la dirección virtual debe traducirse a una dirección física antes de comparar la etiqueta de dirección. En la etapa `s0` de la tubería de control, las direcciones virtuales de las dos solicitudes de línea de caché se envían al puerto de consulta del ITLB al mismo tiempo, y el ITLB devuelve una señal que indica si la dirección virtual llega dentro de este ciclo de reloj. Si se produce un impacto, la dirección física correspondiente se devolverá en el siguiente tiempo. Si ocurre un error, la lógica de control bloqueará la tubería MainPipe y esperará hasta que se vuelva a llenar el ITLB y se devuelva la dirección física.

## Manejo de errores de caché

La solicitud que no se complete se enviará a MissUnit para que envíe una solicitud de adquisición de Tilelink a la caché L2 descendente. Después de que MissUnit reciba la solicitud de concesión para los datos correspondientes, si es necesario reemplazar la línea de caché, MissUnit enviará una solicitud de adquisición de Tilelink a la caché L2 descendente. Solicite a ReplacePipe, y ReplacePipe volverá a leer la SRAM para obtener los datos y luego los enviará a ReleaseUnit para iniciar una solicitud de "Liberación" a la caché L2. Por último, MissUnit rellena la SRAM y devuelve los datos a MainPipe una vez finalizada la recarga. A continuación, MainPipe devuelve los datos a IFU.

Puede ocurrir un error en la línea de caché en cualquiera de las dos solicitudes, por lo que se configuran dos elementos missEntry en MissUnit para manejar los errores y mejorar la simultaneidad.

## Manejo de excepciones
Hay dos tipos principales de excepciones generadas en ICache: Error de página de instrucciones informado por ITLB y Error de acceso informado por ITLB y PMP. MainPipe informa la excepción directamente a la IFU y los datos solicitados se consideran inválidos.

<h2 id=imem>Sección de almacenamiento</h2>

La lógica de almacenamiento de la caché de instrucciones se divide principalmente en Meta SRAM (que almacena la etiqueta y el estado de consistencia de cada línea de caché) y Data SRAM (que almacena el contenido de cada línea de caché). El código de verificación de paridad se admite internamente para la verificación de datos. Cuando se produce un error de verificación, se informará un error de bus y se generará una interrupción. La SRAM de metadatos se divide en bancos de paridad. Dos líneas de caché adyacentes en el espacio de direcciones virtuales se dividirán en bancos diferentes para implementar la lectura de dos líneas de caché a la vez.


## Soporte de consistencia

El caché de instrucciones de la arquitectura Xiangshan Nanhu implementa el protocolo de consistencia definido por Tilelink. Básicamente, se agrega una tubería adicional, ReplacePipe, para manejar las solicitudes `Probe` y `Release` de Tilelink.


El ReplacePipe de la caché de instrucciones consta de una tubería de 4 etapas:

- En la fase `r0`, recibe la solicitud `Probe` enviada desde ProbeUnit y la solicitud `Release` enviada desde MissUnit, y también inicia una lectura de la SRAM Meta/Data. Dado que la solicitud aquí contiene la dirección virtual y la dirección física real, no se requiere traducción de dirección.
- En la fase `r1`, ReplacePipe, al igual que MainPipe, utiliza la dirección física para hacer coincidir la dirección de la línea de caché N-way de un Set devuelto por SRAM, generando señales de aciertos y errores. Esta señal solo es válida para `Probe` , porque `Release` La solicitud debe estar en la caché de instrucciones.
- En la fase `r2`, la solicitud `Probe` de impacto modificará los permisos del bloque de caché correspondiente y, al mismo tiempo, enviará esta solicitud a la ReleaseUnit para enviar una solicitud `ProbrResponse` a L2. Los permisos de esta línea de caché se generan en función de los permisos originales (T/B) y la conversión de permisos de la sonda (toN, toB, toT). Las solicitudes fallidas no realizarán cambios de permiso y se enviarán a ReleaseUnit para informar a L2 que el permiso ha cambiado a NToN (no hay datos solicitados por `Probe` en el caché de instrucciones). También se envía una solicitud 'Release' a ReleaseUnit para enviar 'ReleaseData' a L2. Y solo las solicitudes de `Liberación` pueden ingresar a `r3`
- En la fase `r3`, ReplacePipe informa a MissUnit que el bloque reemplazado ha sido ``liberado`` hacia abajo, notificando a MissUnit que puede rellenarse.


## Precarga de instrucciones

La arquitectura Xiangshan Nanhu implementa un `Fetch Directed Prefetching (FDP)`[^fdp] simple, que permite la predicción de bifurcaciones para guiar la precarga de instrucciones. Para este propósito, se agrega un precargador de instrucciones `IPrefetch`. El mecanismo de precarga específico es el siguiente:

* Se agrega un puntero de precarga a la cola de destino de búsqueda de instrucciones. El puntero se ubica entre el puntero de predicción y el puntero de búsqueda de instrucciones. El puntero de precarga lee la dirección de destino del paquete de instrucciones actual (el destino de salto si se salta, el destino de salto si se salta). no salta). La dirección de inicio del siguiente paquete en secuencia) se envía al prefetcher.
* El buscador completa la traducción de la dirección y accede a la Meta SRAM de la caché de instrucciones. Si descubre que la dirección ya está en la caché de instrucciones, se cancela la solicitud de búsqueda previa. En caso contrario, solicite a `PrefetchEntry` que asigne un elemento, envíe una solicitud `Hint` de Tilelink al caché `L2` y obtenga previamente la línea de caché correspondiente en L2.
* Para garantizar que las solicitudes de prefetch no se envíen repetidamente a L2, el prefetcher registra las direcciones físicas de las solicitudes de prefetch que se han enviado. Cualquier solicitud comprobará este registro antes de solicitar `PrefetchEntry`. Si encuentra una solicitud de prefetch que ha se ha enviado, si es el mismo, la solicitud de prefetch actual se cancelará.

## Referencias
[^fdp]: Reinman G, Calder B, Austin T. Búsqueda previa de instrucciones dirigidas por Fetch[C]//MICRO-32. Actas del 32.º Simposio internacional anual ACM/IEEE sobre microarquitectura. IEEE, 1999: 16-27.

--8<-- "docs/frontend/abreviaturas.md"
