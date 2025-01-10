# TLB de segundo nivel

L2 TLB es un caché de tabla de páginas de mayor capacidad, pero también incluye una unidad Page Table Walker, que es responsable del Hardware Page Table Walk.

## Diseño estructural

La L2 TLB consta de cuatro unidades principales:

1. Caché de páginas: almacena en caché las tablas de páginas. Las tablas de páginas de tres capas de Sv39 se almacenan en caché por separado y la consulta de la información de las tres capas se puede realizar de una sola vez.
2. Page Walker: consulta los dos primeros niveles de las tablas de páginas en la memoria, que es esencialmente una máquina de estados.
3. Last Level Page Walker: consulta la tabla de páginas de último nivel en la memoria
4. Miss Queue: consultas de caché de Page Cache y Last Level Page Walker solicitudes de omisión
5. Prefetcher: Prefetcher

Las solicitudes de la TLB de primer nivel accederán primero a la caché de páginas. Si se encuentra un resultado, se devolverá directamente a la TLB de primer nivel. Si se encuentra un error, ingresará al Page Walker, Last Level Page Walker o Señorita Cola.
La tabla de páginas se puede recorrer a través de Page Walker y Last Level Page Walker; se puede acceder nuevamente al caché de páginas esperando recursos a través de Miss Queue. Después de acceder a la tabla de páginas, puede regresar directamente a la TLB de primer nivel.
El prefetcher supervisa los resultados de la consulta de la caché de páginas y genera solicitudes de prefetch. Los resultados de prefetch se almacenan en la caché de páginas y no se devolverán a la TLB de primer nivel.

## Caché de página

Page Cache es una TLB L2 "real" con una mayor capacidad de caché. Almacena en caché 3 niveles de tablas de páginas y los almacena en caché por separado, de modo que se puede acceder a los 3 niveles de tablas de páginas de una sola vez. Determinar si impacta en función de la dirección virtual y obtener el resultado más cercano al nodo hoja.
La caché de páginas solo almacena entradas válidas de la tabla de páginas. Dado que el ancho de acceso es de 512 bits, es decir, 8 tablas de páginas, una entrada de la caché de páginas contiene 8 tablas de páginas (1 número de página virtual, 8 números de página físicos, 8 bits de permiso).

El acceso a la caché de páginas se divide en cuatro tiempos. El primer tiempo envía una señal de lectura, el segundo tiempo obtiene el resultado de la lectura y lo almacena en caché durante un tiempo, el tercer tiempo determina si se cumple y realiza una comprobación ECC, y el cuarto tiempo devuelve el resultado. .
Si falla una comprobación de ECC, no se enviará ninguna señal de excepción o interrupción al núcleo. En su lugar, se invalidará el elemento actual, se devolverá el resultado de error y se realizará nuevamente el Page Walk.
La caché de página devuelve toda la información que puede, incluyendo:

1. Si el nodo de hoja (incluida la tabla de páginas de 4 KB y la página grande) impacta y la tabla de páginas que impacta se devuelve a la TLB de primer nivel
2. Si el elemento de referencia ha sido precargado por el prefetcher, devuélvaselo al prefetcher para su uso.
3. Si se alcanzan los dos primeros niveles de las tablas de páginas y el resultado de la visita se devuelve al Page Walker o al Last Level Page Walker para su uso

## Pagina Walker

Page Walker es una máquina de estados que accede a la tabla de páginas en función de la dirección virtual. Las reglas de acceso son similares a las del manual de RV hasta que:

1. Después de acceder al nodo de 2 MB, se devuelve al Last Level Page Walker, que luego accede al último nivel de la tabla de páginas.
2. Acceda al nodo hoja, que es una página grande (SuperPage), y devuélvalo a la TLB de primer nivel.
3. Se accede a un nodo ilegal, el bit v de la entrada de la tabla de páginas es falso, etc.

Los resultados del acceso a Page Walker se almacenan en Page Cache.
Page Walker solo puede procesar una solicitud a la vez y solo puede acceder a los dos primeros niveles de las tablas de páginas como máximo. Su capacidad de acceso a la memoria es débil. Las razones se explican en la sección Page Walker de último nivel.

## Página de último nivel Walker

Last Level Page Walker existe para mejorar la capacidad de acceso a la memoria de L2 TLB. Last Level Page Walker no fusiona solicitudes repetidas, sino que las registra y comparte los resultados del acceso a la memoria para evitar accesos repetidos a la memoria.
Last Level Page Walker recibe solicitudes fallidas de Page Cache y los resultados de Page Walk y accede a la tabla de páginas de último nivel.
Si a la solicitud de la caché de páginas solo le falta la tabla de páginas de último nivel, se puede acceder a la memoria. De lo contrario, se debe acceder nuevamente a la caché de páginas hasta obtener la dirección de la tabla de páginas de último nivel.
La solicitud de Page Walker puede acceder a la memoria porque ha obtenido los dos primeros niveles de tablas de páginas y solo le falta el último nivel de tabla de páginas.

Last Level Page Walker y Page Walker trabajan juntos para completar todo el proceso de Page Table Walk. Para mejorar el paralelismo del acceso a la memoria, Last Level Page Walker configura diferentes ID para las solicitudes y tiene múltiples solicitudes en curso al mismo tiempo. Sin embargo, los dos primeros niveles (1 GB, 2 MB) de diferentes solicitudes pueden ser los mismos, y considerando que la probabilidad de error de los dos primeros niveles es menor que la de la tabla de páginas del último nivel, no consideramos mejorar el paralelismo de acceso de los dos primeros niveles de tablas de páginas y solo establecer un Page Walk reducen la complejidad del diseño.

## Señorita cola

La esencia de Miss Queue es una cola de búfer que sirve para esperar recursos. La cola de señoritas puede recibir solicitudes de la caché de página o del último nivel de la página Walker, esperando acceder a la caché de página nuevamente.

## Precarga

Actualmente, se utiliza el algoritmo de precarga NextLine. Cuando se produce un error o un acierto, pero el elemento acertado es un elemento de precarga, se genera la siguiente solicitud de precarga.
Las solicitudes de precarga acceden a la caché de páginas. Debido a que el Page Walker tiene capacidades de acceso a la memoria débiles, si una solicitud de precarga falla, no ingresará al Page Walker ni a la cola de fallas, sino que se descartará directamente. Si a la solicitud de precarga solo le falta la última sección de la tabla de páginas, se puede acceder al Last Level Page Walker.
Los resultados de la búsqueda previa se almacenan en la caché de página y no se devuelven a la TLB de primer nivel.
