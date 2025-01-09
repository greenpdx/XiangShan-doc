# Nivel 1 TLB

## Introducción básica

TLB (Translation Lookaside Buffer) es responsable de la traducción de direcciones.
Xiangshan admite el modo Sv39 especificado en el manual, pero actualmente no admite otros modos. La longitud de la dirección virtual es de 39 bits y la longitud de la dirección física es de 36 bits, que se pueden modificar mediante parámetros.
La habilitación de la memoria virtual depende del nivel de privilegio actual (como el modo M o el modo S) y del registro Satp. Esta determinación se realiza dentro de la TLB y es transparente para el exterior de la misma. Por lo tanto, desde la perspectiva de los módulos Fuera de la TLB, todas las direcciones se traducen mediante la TLB.

## Forma organizativa

Antes de acceder a la memoria dentro del núcleo, tanto la obtención de instrucciones del front-end como el acceso a la memoria del back-end deben ser traducidos por el TLB. Debido a la gran distancia física y para evitar la contaminación mutua, la ITLB de front-end (TLB de instrucciones) y la DTLB de acceso a memoria de back-end (TLB de datos) son dos TLB. ITLB utiliza un modo totalmente asociativo, con 32 páginas normales (NormalPage) responsables de páginas de 4 KB y 8 páginas grandes (SuperPage) responsables de la traducción de direcciones de 2 MB y 1 GB. DTLB adopta un modo híbrido, con 64 páginas normales directamente asociadas para ser responsables de páginas de 4 KB y 16 páginas grandes totalmente asociadas para ser responsables de páginas de todos los tamaños, utilizando una estrategia de reemplazo pseudo LRU. La parte de página normal proporciona capacidad, pero tiene poca flexibilidad y es propensa a conflictos; la parte de página grande tiene alta flexibilidad pero poca capacidad. Por lo tanto, la parte de página normal se utiliza como víctima de la parte de página grande. Al rellenar, la parte de página grande se rellena primero y los elementos expulsados ​​de la parte de página grande se escriben en la parte de página normal (solo páginas de 4 KB).

El acceso a la memoria tiene dos canales de carga y dos canales de almacenamiento y, por cuestiones de tiempo, requieren una TLB independiente. Al mismo tiempo, por razones de rendimiento, el contenido de estas TLB debe permanecer consistente, de manera similar a la precarga mutua. Por lo tanto, estos cuatro DTLB utilizan un módulo de algoritmo de reemplazo “externo” para garantizar la consistencia del contenido.

## Bloqueo y no bloqueo

La búsqueda de instrucciones del front-end requiere bloquear el acceso a la ITLB, es decir, cuando la TLB falla, esta no devuelve inmediatamente el resultado de la falla, sino que realiza un recorrido de tabla de páginas para recuperar la entrada de la tabla de páginas y luego regresa.
El acceso a la memoria del back-end requiere acceso sin bloqueo al DTLB, es decir, cuando el TLB falla, el TLB también debe devolver el resultado inmediatamente, ya sea un error o un acierto.
En la arquitectura Yanqihu y la arquitectura Nanhu, el cuerpo de la TLB es un acceso sin bloqueo, el acceso de bloqueo de la búsqueda de instrucciones del front-end es reproducido por el módulo front-end y la TLB no almacena la información solicitada.

Vista previa de nueva arquitectura: TLB puede parametrizar la forma de bloqueo de cada puerto de solicitud, seleccionar estáticamente bloqueo o no bloqueo, y un módulo puede adaptarse a las necesidades tanto del front-end como del back-end.

## Sfence.vma y ASID

Cuando se ejecuta la instrucción Sfence.vma, se borra todo el contenido del búfer de almacenamiento (se vuelve a escribir en DCache) y luego se envía una señal de actualización a varias partes de la MMU. La señal de actualización es unidireccional y solo dura un tiempo, sin señal de retorno. La instrucción finalmente actualizará todo el pipeline y lo volverá a ejecutar desde la búsqueda de instrucciones.
Sfence.vma cancela todas las solicitudes en vuelo, incluidas las de repetidor y filtro, así como las solicitudes en vuelo en L2 TLB, y actualiza las tablas de páginas en caché en TLB y L2TLB según la dirección y el ASID.
La parte totalmente asociativa se actualiza en función de si se golpea o no, y la parte asociativa del grupo (directamente asociativa) se actualiza directamente en función del índice.

La arquitectura Yanqihu no admite ASID (identificador de espacio de direcciones). La arquitectura Nanhu añade compatibilidad con ASID, que tiene una longitud de 16 y se puede parametrizar.
No todas las solicitudes durante el vuelo contienen información ASID. Esto se debe a las siguientes razones:

1. Generalmente, el ASID de la solicitud en vuelo es el mismo que el campo ASID de Satp
2. Al cambiar de ASID, las solicitudes en curso son todas accesos "especulativos", que son casi "inútiles".

Combinando estas dos razones, al cambiar el ASID, todos los vuelos a bordo también se cancelan y el vuelo no necesita llevar información del ASID.

## Actualizar bits A/D

Según el manual, será necesario actualizar los bits A (Acceso) y D (Sucio) de la tabla de páginas. Xiangshan utiliza el método de informar una excepción de error de página y los actualiza mediante software.
