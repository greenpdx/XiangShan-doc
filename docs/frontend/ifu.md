# Documentación de la unidad de obtención de instrucciones
![ifu](../figs/frontend/IFU.png)

En este capítulo se describe la implementación de la unidad de búsqueda de instrucciones (IFU) del procesador Xiangshan. La secuencia de comandos de la unidad de búsqueda de instrucciones se muestra en la figura anterior.

La IFU de la arquitectura Nanhu adopta una estructura de canalización de 4 etapas, que es mucho más simple que el diseño de la IFU de la versión Yanqi Lake. Esto se debe al uso de una arquitectura de búsqueda de instrucciones desacoplada de caché de instrucciones y predicción de ramificaciones.

Después de que se emite una solicitud de obtención de instrucciones desde el FTQ, pasa por las siguientes etapas en la IFU:

- La solicitud de búsqueda de instrucciones enviada desde FTQ contiene un código de instrucción de 32 bytes (llamado bloque de instrucciones) y la dirección inicial y la dirección del próximo objetivo de salto.
- Enviar solicitudes tanto al pipeline IFU como al módulo ICache en la etapa `IFU0`.
- La etapa `IF1` realiza algunos cálculos simples (por ejemplo, cada 2 bytes en este bloque de instrucciones, es decir, el PC de cada instrucción posible).
- En la etapa `IF2`, después de esperar a que el caché de instrucciones devuelva hasta dos líneas de caché de datos (porque puede haber una situación en la que el [bloque de instrucciones cruza líneas](#crossfetch)), el primer paso es dividir la instrucción y obtener la dirección de la instrucción. Los códigos de comando fuera del rango válido se descartan. El comando se envía al predecodificador para la predecodificación y el comando comprimido de 16 bits se expande a un comando de 32 bits.
- La etapa `IF3` enviará primero el resultado de pre-descodificación al [verificador de predicción de ramificación](#predchecker). Si se encuentra un error, la secuencia de IFU se actualizará en el siguiente latido y la información se enviará al FTQ para actualizar el predictor y volver a recuperar la instrucción. Los buffers en los que no se han detectado errores esperan ser decodificados en la cola de buffer de instrucciones (IBuffer).
- La etapa `IF3` también iniciará una solicitud de búsqueda de instrucciones al módulo MMIO de instrucciones en función del resultado de la traducción de direcciones y, al mismo tiempo, cambiará al [modo de búsqueda de instrucciones MMIO](#mmiofetch) y ejecutará las instrucciones una por una. secuencialmente.
- La lógica de control IFU también necesita [manejar el caso de la mitad de una instrucción RVI](#half).

## Pre-descodificación

El predecodificador decodifica los 16 códigos de instrucciones de 16 bits después de la segmentación para obtener información sobre la instrucción (si se trata de una instrucción de salto, tipo de instrucción de salto, si se trata de una instrucción comprimida, etc.). Para la instrucción de salto, también calcula Su dirección de destino. El objetivo principal es proporcionar información de instrucciones y la dirección de destino correcta al [verificador de predicción de ramas](#predchecker) y actualizar la información de instrucciones en el predictor de manera oportuna.

Por otro lado, el predecodificador también expandirá las instrucciones comprimidas (si hay alguna en este bloque de instrucciones) en instrucciones de 32 bits de longitud para simplificar la lógica de decodificación posterior.

## Comprobación de predicción de rama

Después de obtener la información de predecodificación de la instrucción, el verificador de predicción de bifurcación verifica principalmente los siguientes errores:

- Error de predicción de instrucción de salto: verifique las dos instrucciones, jal y ret, que están destinadas a saltar. Si estas dos instrucciones están en el rango de instrucciones válidas de este bloque y la predicción es Si no salta, se considera un Error de predicción.
- Predicción errónea de instrucciones que no son de salto: si la instrucción predicha como un salto no está en el rango de instrucciones válidas, o está en el rango de instrucciones válidas pero no es una instrucción de salto, se considera una predicción errónea.
- Error de dirección de destino: para instrucciones de salto cuya dirección de destino se puede conocer a partir del código de instrucción (excepto las instrucciones de salto `jalr`), si está dentro del rango de instrucción válido y se predice el salto y la dirección de destino del salto no coincide con la dirección correcta, se considera un error de predicción.

Después de descubrir un error, el verificador de predicción de bifurcaciones selecciona la instrucción de error predicha al principio de la secuencia de instrucciones, pasa la información de error (la ubicación de la instrucción de error en el bloque, la información de predecodificación de la instrucción y la dirección de destino correcta) a FTQ y despeja el oleoducto IFU. La IFU espera que el FTQ vuelva a enviar la solicitud de búsqueda de instrucciones.

## Búsqueda entre líneas

Dado que nuestro bloque de instrucciones incluye 32 bytes de código de instrucción, lo que equivale al tamaño de la mitad de una línea de caché (64 bytes), si la dirección de inicio de este bloque está en la segunda mitad de la línea de caché, es totalmente posible que el El rango de bloques se cruzará. Por lo tanto, la caché de instrucciones admite la obtención de dos líneas de caché a la vez para garantizar el rendimiento de las instrucciones en este caso. El enfoque específico es que cuando FTQ descubre que la dirección inicial del bloque está en la segunda mitad de la línea de caché, inicia una solicitud para dos líneas de caché adyacentes del caché de instrucciones.

## Rango de comando válido

El rango efectivo de un bloque de instrucción de búsqueda está determinado por la dirección de inicio dada por FTQ y el índice de la instrucción de salto (si hay un salto). Si no hay instrucción de salto en este bloque, el rango efectivo de instrucción predeterminado es el de inicio. Dirección. 256 bits.

El alcance de las instrucciones válidas podrá ser determinado nuevamente por la inspección de la IFU, incluyendo principalmente:

* Cuando la verificación de predicción de rama encuentra instrucciones de salto `jal` y `ret` no previstas, es necesario acortar el rango de instrucciones válidas a la primera de dichas instrucciones de salto;
* En el caso donde el bloque anterior tiene medio RVI, los primeros 16 bits del bloque siguiente no están dentro del rango de instrucciones válidas.
* El rango efectivo de la instrucción de bloque de la solicitud MMIO es de solo 32 bits

## Obtención de instrucción MMIO

En la etapa `IF3`, si ITLB encuentra que la dirección está en el espacio MMIO, la IFU inicia el modo de búsqueda de instrucciones MMIO y envía una solicitud al módulo MMIO de instrucciones. El módulo MMIO de instrucciones envía una solicitud `Get` para 64 bits de datos al bus MMIO y espera a que el bus regrese. Luego, los datos se recortan según la dirección de solicitud de la IFU y se devuelve el código de instrucción. La IFU expande el código de instrucción y lo envía a la cola del búfer de instrucciones. Al mismo tiempo, la IFU bloquea la tubería y escucha la señal de confirmación del ROB hasta que se ejecuta la instrucción y envía una redirección del frontend para buscar la siguiente instrucción.

Las solicitudes MMIO obtienen solo una instrucción a la vez, por lo que la velocidad de ejecución de instrucciones del procesador se vuelve muy lenta en este modo.

## Procesamiento de la mitad de una instrucción RVI

Cuando se encuentra un bloque de instrucciones en la etapa `IF3` cuyos últimos 2 bytes son la primera mitad de una instrucción RVI, contamos esta instrucción RVI en este bloque y utilizamos el mecanismo de tomar dos líneas de caché para asegurarnos de que la segunda la mitad es Definitivamente se puede recuperar, por lo que solo necesitamos establecer una bandera cuando esto sucede y excluir los primeros 2 bytes del rango válido de la instrucción cuando llega el siguiente bloque.

--8<-- "docs/frontend/abreviaturas.md"
