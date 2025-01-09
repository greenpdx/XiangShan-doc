# Arquitectura general de la caché L2/L3

La caché L2/L3 de la arquitectura XiangShan Nanhu (es decir, el [subproyecto huancun](https://github.com/OpenXiangShan/HuanCun)) se basa en [block-inclusive-cache-sifive](https://github.com/sifive / block-inclusivecache-sifive) es un caché no inclusivo basado en directorios (directorio inclusivo, datos no inclusivos) diseñado.

huancun utiliza Tilelink como protocolo de consistencia de bus y puede resolver el [problema de alias de caché](./cache_alias.md) cuando el caché L1 es mayor a 32 KB agregando un bit de usuario Tilelink personalizado.

La estructura general de huancun se muestra en la siguiente figura:
![](../figs/huancun.png)

Huancun puede dividir el índice según los bits bajos de la dirección solicitada para mejorar la concurrencia. La cantidad de [MSHR](./mshr.md) dentro de cada Slice es configurable y es responsable de la gestión de tareas específicas.

[DataBanks](./data.md) se encarga de almacenar datos específicos. La cantidad de bancos se puede configurar mediante parámetros para mejorar el paralelismo de lectura y escritura.

[RefillBuffer](./misc.md#refill_buffer) es responsable de almacenar temporalmente los datos de recarga para que puedan enviarse directamente a la memoria caché de nivel superior sin escribirse en la SRAM.

Sink/Source\* El módulo relevante es el Tilelink [módulo de control de canal](./channels.md), que es responsable de interactuar con la interfaz estándar de Tilelink, convirtiendo las solicitudes externas en
Las señales internas de caché, por otro lado, reciben solicitudes internas de caché y las convierten en solicitudes Tilelink y las envían a la interfaz.

En la [organización de directorios](./directory.md), huancun almacena los directorios de los datos de la capa superior y los datos de la capa actual por separado.
El directorio propio/directorio del cliente son los directorios correspondientes a los datos de caché de nivel actual y a los datos de caché de nivel superior respectivamente.

Además, el [prefetcher](./prefetch.md) utiliza el algoritmo BOP (Best-Offset Prefetching), que se puede configurar o recortar mediante parámetros.



## El flujo de trabajo general de huancun es:

1. El [módulo de control de canal](./channels.md) acepta solicitudes de Tilelink y las convierte en solicitudes de caché interna.

2. El [módulo Alloc MSHR](./misc.md#alloc) asigna un [MSHR](./mshr.md) para la solicitud interna.

3. [MSHR](./mshr.md) inicia distintas tareas según los requisitos de las distintas solicitudes. Los tipos de tareas incluyen la lectura y escritura de datos, el envío de nuevas solicitudes a las cachés superior e inferior o la devolución de respuestas, la activación o actualización de prefetchers, etc.

4. Cuando se completan todas las operaciones necesarias para una solicitud en [MSHR](./mshr.md), [MSHR](./mshr.md) se libera y espera nuevas solicitudes.
