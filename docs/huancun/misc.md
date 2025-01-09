# Diseño de búfer de solicitud

Para evitar la competencia de recursos y la interferencia mutua cuando se procesan múltiples solicitudes simultáneas en la memoria caché al mismo tiempo, huancun bloquea las solicitudes en la granularidad del conjunto.
Las solicitudes de la misma prioridad (A, B y C) y el mismo conjunto no pueden ingresar a MSHR al mismo tiempo. Para reducir la pérdida de rendimiento de la memoria caché causada por esta estrategia de bloqueo, diseñamos el búfer de solicitudes para almacenar en buffer las solicitudes bloqueadas, de modo que las solicitudes posteriores de diferentes conjuntos puedan ingresar al MSHR sin bloquearse.

Teniendo en cuenta las características de la cantidad de solicitudes del bus Tilelink A>>B, A>>C en la CPU, Request Buffer solo almacena en búfer la solicitud del canal A, de modo que
Mejore el rendimiento y reduzca la complejidad de implementación del hardware.

El diseño del búfer de solicitudes es similar a la estación de reserva/cola de transmisión en la CPU. Cuando una nueva solicitud no puede ingresar al MSHR, intentará ingresar al búfer de solicitudes. Si hay un elemento vacío en el búfer, la solicitud se enviará a la estación de reserva. se asignará a la posición correspondiente y se registrará la solicitud. Qué MSHR se está esperando para ser liberado (`wait_table` en el código). Cuando se libera MSHR, su identificación MSHR se transmitirá al Buffer de solicitudes y los elementos en el Buffer que tienen una dependencia en MSHR se pueden "despertar".

Cuando hay varias solicitudes con el mismo conjunto en el búfer de solicitudes, para garantizar que estas solicitudes puedan salir del búfer en orden FIFO, el búfer interno
También se mantiene una **matriz de dependencia** `buffer_dep_mask`, que registra las dependencias entre elementos en el Buffer.
`buffer_dep_mask[i][j]` es 1, lo que significa que la solicitud en la posición `i` y la solicitud en la posición `j` son del mismo conjunto y están ubicadas en la posición `i`.
La solicitud en la posición `j` llegó más tarde que la solicitud en la posición `j`, por lo que la solicitud `j` debe salir del Buffer primero cuando se libera el MSHR externo.



# Diseño de buffer de recarga

Para reducir la latencia de Cache Miss, Huancun utiliza Refill Buffer para almacenar en búfer los datos rellenados desde el caché o la memoria de nivel inferior.
De esta manera, los datos rellenados se pueden devolver directamente a la memoria caché de nivel superior sin tener que escribirse primero en SRAM.



# MSHR Alloc (módulo de asignación MSHR) <a name="alloc"></a>

Según la especificación manual de Tilelink, para evitar bloqueos en el sistema, las solicitudes de alta prioridad deben poder interrumpir las solicitudes de baja prioridad y ejecutarse primero.
Por lo tanto, huancun diseñó N (N >= 1) MSHR abc, 1 MSHR b y 1 MSHR c.

En ausencia de conflictos de conjuntos, todas las solicitudes eligen una entrada vacía en el MSHR abc;
Cuando una solicitud de alta prioridad recién llegada entra en conflicto con una solicitud de baja prioridad que ya está en el MSHR, la solicitud de alta prioridad ingresa al MSHR dedicado.
Por ejemplo, si la solicitud b entra en conflicto con la solicitud a en abc MSHR, la nueva solicitud b se asignará a b MSHR.
Si la solicitud c entra en conflicto con la solicitud a/b en el MSHR abc/b, la nueva solicitud c se asignará al MSHR c.



# Ayudante de sonda

Dado que huancun adopta el diseño de directorio inclusivo de datos no inclusivos, cuando el Directorio del Cliente
Cuando la capacidad es limitada y no se puede almacenar un nuevo bloque de caché (por ejemplo, bloque A), la caché de este nivel debe enviar una solicitud de sonda al nivel superior para transferir un bloque de caché del nivel superior al bloque de caché.
Se prueba el bloque de caché (por ejemplo, bloqueB) y luego el estado correspondiente del nuevo bloque de caché (bloqueA) se almacena en el directorio del cliente.

Para simplificar el procesamiento de un solo MSHR, diseñamos ProbeHelper para monitorear los resultados de lectura del Directorio de clientes.
Si hay un conflicto de capacidad, ProbeHelper genera una solicitud B falsa para bajar el bloque de destino desde la sonda de nivel superior.
De esta manera, no es necesario considerar el conflicto de capacidad de Client Directory en un solo MSHR, lo que simplifica el diseño.
