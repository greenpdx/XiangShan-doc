# Implementación de instrucciones atómicas

Este capítulo presenta el flujo de procesamiento de instrucciones atómicas de la arquitectura Nanhu del procesador Xiangshan.

!!! nota
 El flujo de procesamiento de instrucciones atómicas puede cambiar en la próxima versión

## Proceso de ejecución de instrucciones atómicas

Las instrucciones atómicas de la arquitectura Southlake están controladas por una máquina de estados independiente. Una instrucción atómica pasa por los siguientes estados:

* s_invalid: máquina de estados inactiva
* s_tlb: la instrucción atómica emite una solicitud de consulta TLB
* s_pm: realiza la búsqueda de TLB y la verificación de excepciones de dirección
* s_flush_sbuffer_req: emite una solicitud de vaciado de sbuffer/sq
* s_flush_sbuffer_resp: esperar a que se complete la solicitud de vaciado de sbuffer/sq
* s_cache_req: inicia la operación de instrucción atómica en dcache
* s_cache_resp: espera a que dcache devuelva el resultado de la instrucción atómica
* s_finish: escribe el resultado nuevamente

Para una instrucción de tienda, el procesamiento solo comenzará cuando su dirección y datos estén listos.

## Soporte de dcache para instrucciones atómicas

Actualmente, todas las instrucciones atómicas de Xiangshan (arquitectura Nanhu) se procesan en dcache. dcache utiliza la tubería principal (MainPipe) para ejecutar instrucciones atómicas. Si MainPipe detecta que la instrucción atómica falla, enviará una solicitud a MissQueue. Después de que MissQueue obtenga los datos desde la caché l2 y luego ejecute la instrucción atómica. Si MissQueue está llena cuando se ejecuta la instrucción atómica, seguirá intentando reenviar la instrucción atómica hasta que MissQueue reciba la solicitud de reabastecimiento por error de la instrucción atómica. El dcache utiliza un elemento por elemento `AtomicsReplayEntry` para programar la instrucción atómica.

### Soporte especial para lr/sc

La especificación RISC-V define bucles LR/SC restringidos (consulte Volumen I: RISC-V Unprivileged ISA: 8.3 Eventual Success of Store-Conditional Instructions). Inspirado por Rocket, Xiangshan proporciona un mecanismo similar para manejar bucles LR/SC restringidos.

Después de ejecutar la instrucción lr, el procesador ingresará a los siguientes tres estados en secuencia:

1. Bloquear todas las solicitudes de sondeo a la línea de caché actual del núcleo actual y bloquear la ejecución de lr subsiguientes durante un período de tiempo (ciclos `lrscCycles` - `LRSCBackOff`)
1. Continuar bloqueando la ejecución de LR posteriores, pero permitir que las solicitudes de sondeo para esta línea de caché se ejecuten durante un período de tiempo (ciclos `LRSCBackOff`)
1. Volver al estado normal

Para Xiangshan (Nanhu), el bucle LR/SC restringido (hasta 16 instrucciones) debe ejecutarse dentro del ciclo `lrscCycles` - `LRSCBackOff`. Durante este tiempo, se sondea la dirección en el conjunto de reserva establecido por LR. La operación se bloqueará. El núcleo actual puede realizar una SC exitosa sin interrupción. Los LR subsiguientes no se ejecutarán durante el ciclo `LRSCBackOff` subsiguiente para evitar que dos núcleos ejecuten LR al mismo tiempo, lo que provocaría que ninguno pueda aceptar solicitudes de sondeo. Ninguno de ellos puede obtener el permiso y, por lo tanto, se quedan bloqueados. Después de agregar esta fase de retroceso, las solicitudes de sondeo de otros núcleos pueden obtener permisos de línea de caché.

### Soporte para el comando amo

dcache realizará operaciones definidas por las directivas amo en mainpipe. Consulte la sección [dcache mainpipe](../dcache/main_pipe.md#stage-2).

## Manejo de excepciones para instrucciones atómicas

Cuando el mecanismo de verificación de direcciones detecta una excepción, la máquina de estado de instrucciones atómicas ingresará directamente `s_finish` desde `s_pm`, sin activar las operaciones de cola de almacenamiento y actualización del búfer, y sin generar solicitudes de acceso a memoria reales. El dcache no estará al tanto de La instrucción atómica.

## Relacionado con la depuración

El disparador de instrucciones atómicas solo admite el uso del disparador vaddr.
