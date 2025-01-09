# Cola de destino de obtención (FTQ)
Este capítulo describe la implementación de la cola de destino de obtención del procesador Xiangshan.
<figura rebajada>
 ![ftq](../figs/frontend/ftq.svg){ ancho="700" }
 <figcaption>Interacción entre FTQ y otros módulos</figcaption>
</figura>

## Descripción del trabajo
FTQ es una cola intermedia entre las unidades de predicción de bifurcación y de obtención de instrucciones. Su función principal es almacenar temporalmente los objetivos de obtención de instrucciones predichos por la BPU y enviar solicitudes de obtención de instrucciones a la IFU en función de estos objetivos de obtención de instrucciones. Otra función importante es almacenar temporalmente la información de predicción de cada predictor de BPU y enviar esta información de vuelta a BPU para el entrenamiento del predictor después de que se envíe la instrucción. Por lo tanto, necesita mantener la instrucción desde la predicción hasta el envío. Ciclo de vida completo **. Dado que la PC de almacenamiento del backend tiene una gran sobrecarga, cuando el backend necesita dar instrucciones a la PC, acudirá a FTQ para leerlas.

Estructura interna
FTQ es una estructura de cola, pero el contenido de cada elemento de la cola se almacena en una estructura de almacenamiento diferente según sus propias características. Estas estructuras de almacenamiento incluyen principalmente lo siguiente:

- **ftq_pc_mem**: implementación de la pila de registros, almacena información relacionada con las direcciones de instrucciones, incluidos los siguientes campos

 - `startAddr` predice la dirección de inicio del bloque
 - `nextLineAddr` predice la dirección inicial de la siguiente línea de caché del bloque
 - `isNextMask` predice si cada posible posición de inicio de instrucción del bloque está en la siguiente región alineada con el ancho previsto
 - `fallThruError` predice si la siguiente dirección de búsqueda de instrucción secuencial es incorrecta

- **ftq_pd_mem**: Implementación de la pila de registros, almacena la información de decodificación de cada instrucción en el bloque de predicción devuelto por la unidad de búsqueda de instrucciones, incluidos los siguientes campos

 - `brMask` Si cada instrucción es una instrucción de bifurcación condicional
 - `jmpInfo` predice información sobre la instrucción de salto incondicional al final del bloque, incluyendo si existe, si es un `jal` o un `jalr`, y si es una instrucción `call` o `ret`
 - `jmpOffset` predice la ubicación de la instrucción de salto incondicional al final del bloque
 - `jalTarget` predice la dirección de salto de `jal` al final del bloque
 - `rvcMask` si cada instrucción es una instrucción comprimida

- **ftq_redirect_sram**: implementación de SRAM, almacena información de predicción que debe restaurarse durante la redirección, incluida principalmente información relacionada con RAS y el historial de ramas

- **ftq_meta_1r_sram**: implementación de SRAM para almacenar el resto de la información de predicción de BPU

- **ftb_entry_mem**: implementación del archivo de registro, almacena la información necesaria de la entrada FTB durante la predicción y se utiliza para entrenar nuevas entradas FTB después del envío

Además, cierta información como los punteros de cola y el estado de cada elemento de la cola se implementa mediante registros.


## Ciclo de vida de las instrucciones en FTQ
Las instrucciones se envían a FTQ en unidades de [bloques de predicción](./bp.md#pred-block) después de ser predichas por la BPU hasta que la instrucción está en el [bloque de predicción](./bp.md#pred-block). . Después de que se envíen todas las instrucciones en el backend, FTQ liberará por completo los elementos correspondientes al [bloque de predicción](./bp.md#pred-block) en la estructura de almacenamiento. Esto es lo que sucede durante este proceso:

1. El bloque de predicción se envía desde la BPU y entra en el FTQ. El puntero `bpuPtr` se incrementa en uno, se inicializan los distintos estados de los elementos FTQ correspondientes y se escribe información de predicción en la estructura de almacenamiento. Si la predicción El bloque proviene de la lógica de predicción de sobrescritura de BPU, se restaura el puntero `bpuPtr`. ` y `ifuPtr`
2. FTQ envía una solicitud de búsqueda de instrucciones a IFU, el puntero ifuPtr aumenta en 1 y espera a que se vuelva a escribir la información predecodificada.
3. La IFU vuelve a escribir la información de predescodificación y el puntero `ifuWbPtr` se incrementa en uno. Si la predescodificación detecta un error de predicción, se envía una solicitud de redirección correspondiente a la BPU para restaurar `bpuPtr` e `ifuPtr`.
4. La instrucción ingresa al backend para su ejecución. Si el backend detecta una predicción errónea, notifica a FTQ y envía una solicitud de redireccionamiento a IFU y BPU para restaurar `bpuPtr`, `ifuPtr` e `ifuWbPtr`
5. La instrucción se envía al backend y se notifica al FTQ. Una vez que se han enviado todas las instrucciones válidas en el elemento FTQ, el puntero `commPtr` se incrementa en uno y la información correspondiente se lee desde la estructura de almacenamiento y se envía. a la BPU para entrenamiento.

El ciclo de vida de las instrucciones en el bloque de predicción `n` involucra cuatro punteros en FTQ: `bpuPtr`, `ifuPtr`, `ifuWbPtr` y `commPtr`. Cuando `bpuPtr` comienza a apuntar a `n+1`, el bloque de predicción Las instrucciones en el bloque de predicción ingresan al ciclo de vida, y cuando `commPtr` apunta a `n+1`, las instrucciones en el bloque de predicción completan el ciclo de vida.

## Otras características de FTQ
- Dado que la BPU es básicamente no bloqueante, a menudo puede adelantarse a la IFU, por lo que las solicitudes de búsqueda de instrucciones proporcionadas por la BPU que aún no se han enviado a la IFU se pueden usar como búsquedas previas de instrucciones. Esta parte de la lógica es Implementado en FTQ, entregando directamente la caché de instrucciones. Envío de una solicitud de precarga.
- FTQ agrega la mayor parte de la información del front-end, por lo que implementa muchos contadores de rendimiento que se pueden obtener durante la simulación. Para obtener más detalles, consulte el [código fuente](https://github.com/OpenXiangShan/XiangShan/blob/20bb5c4c094f06264df0e406d0df058f04ccc21c/ fuente/principal/scala/xiangshan/frontend/NewFtq.scala#L1024-L1206)

--8<-- "docs/frontend/abreviaturas.md"
