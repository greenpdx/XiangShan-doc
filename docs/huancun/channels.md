# Control de canal Tilelink

Este capítulo presentará la estructura de cada módulo de control de canal en huancun. Familiarícese con el protocolo de bus TileLink antes de leer.

El módulo de control de canal se divide en el módulo Sink y el módulo Source, entre los cuales,

* El módulo Sink recibe solicitudes activas y respuestas pasivas en el bus TileLink. Para las solicitudes activas (SinkA, SinkB, SinkC), convierta la solicitud en una solicitud interna de huancun y envíela al módulo Alloc de MSHR o al búfer de solicitudes. Para las respuestas pasivas, procese la respuesta y envíela de vuelta al MSHR correspondiente (SinkD, SinkE). ).

* El módulo Fuente recibe la solicitud interna de MSHR, la procesa, la empaqueta y la envía al bus TileLink (FuenteA, FuenteB, FuenteC, FuenteD, FuenteE)

Además, algunos módulos también recibirán tareas adicionales enviadas desde MSHR para completar algunas tareas relacionadas.



## FregaderoA

SinkA reenvía la solicitud recibida del canal A a MSHR Alloc. Si no hay datos, se reenvía directamente; si hay datos (como PutData, etc.), se deben almacenar en PutBuffer. Cada latido de datos se almacena en el elemento libre de PutBuffer en forma de matriz, y el índice del elemento libre se envía a MSHR Alloc.

Los datos almacenados en PutBuffer serán utilizados por SourceD o SourceA. Cuando la solicitud Put llega al caché, SourceD es responsable de escribirla en DataStorage; cuando falla, SourceA es responsable de reenviarla directamente al siguiente nivel de caché o memoria.

Al recibir una nueva solicitud, el canal A se bloqueará porque MSHR Alloc no está listo o PutBuffer está lleno (cuando hay datos); una vez que se recibe una solicitud con datos, la recepción de Beats posteriores no se bloqueará.



## FregaderoB

SinkB reenvía la solicitud recibida del canal B a MSHR Alloc y no tiene otra lógica.



## FregaderoC

SinkC recibe solicitudes de la capa superior para liberar permisos o datos, incluidas las respuestas Release/ReleaseData liberadas activamente por el Cliente y las respuestas ProbeAck/ProbeAckData liberadas pasivamente, y reenvía la solicitud a MSHR Alloc.

El procesamiento de ReleaseData/ProbeAckData es similar al flujo de procesamiento de SinkA para la solicitud PutData. SinkC mantiene un Buffer, y cada Beat de los datos se almacena en el elemento libre del Buffer en forma de matriz, y el índice se envía a MSHR Alloc. SinkC recibe tareas de MSHR para procesar los datos del búfer. Las tareas se dividen en tres tipos:

* Guardar: Almacena los elementos de datos del búfer en DataStorage
* A través de: Liberar los elementos de datos en el búfer directamente a la memoria caché o memoria de nivel inferior después del empaquetado
* Drop: descartar el elemento de datos en el búfer

Al recibir una tarea, SinkC entra en el estado Ocupado y no recibe tareas posteriores hasta que se procesa la tarea. Si la tarea no ha recibido los datos Beat que necesita procesarse, se bloqueará (controlado por beatValsThrough/beatValsSave).



## FregaderoD

SinkD recibe las respuestas Grant/GrantData y ReleaseAck del canal D, utiliza el campo Fuente para consultar a MSHR para obtener el conjunto y la forma, y ​​devuelve la señal Resp a MSHR cuando recibe el primer o el último Beat. Para las respuestas con datos, los datos concedidos se enviarán tanto a DataStorage como a [RefillBuffer](misc.md).



## Hundir

Recibe GrantAck del canal E y envía Resp a MSHR. Aparte de eso, no hay otra lógica.



FuenteA

Recibe solicitudes de adquisición y colocación de MSHR y las reenvía a través del canal A. Para las solicitudes Put, los datos se leen desde PutBuffer en SourceA y luego se reenvían hacia abajo.



## FuenteB

SourceB recibe la tarea de MSHR y envía una solicitud de sonda a través del canal B. SourceB mantiene un registro de estado workVec internamente. Cada bit de workVec corresponde a un cliente que admite Probe. Cuando workVec está vacío, puede recibir solicitudes de MSHR y marcar todos los clientes que necesitan Probe en el registro workVec. Los sondeos se envían a estos clientes en gira y los bits de marca se borran a su vez.



FuenteC

SourceC recibe la tarea de MSHR y es responsable de enviar una solicitud de lectura a DataStorage. Después de recibir los datos, los almacena en la cola y los envía desde el canal C después de que se los saca de la cola.

El motivo para configurar una cola es que la canalización de SourceC no tiene un mecanismo de bloqueo y DataStorage no tiene funciones de bloqueo y cancelación. Esto requiere que una vez que se permite recibir una tarea, se la debe poder procesar e ingresar en La cola. Aquí se utiliza el control de contrapresión para garantizar que, incluso si el canal C está siempre bloqueado, el espacio en la cola puede acomodar todos los elementos que puedan ingresar: incluidos los elementos SRAMLatency inminentes en la tubería más el beatSize obtenido por la tarea actual para leer dataStorage. Artículo.



## FuenteD

SourceD es el módulo de control de canal más complejo. Tiene dos responsabilidades principales: leer datos y enviarlos de vuelta al superior a través del canal D, y enviar solicitudes de escritura a DataStorage para procesar solicitudes Put y otras solicitudes.

El nivel 1 envía una solicitud de lectura a DataStorage o RefillBuffer en función de la información de la tarea y configura Busy para bloquear nuevas tareas. Las nuevas tareas solo pueden ingresar después de que se envíe la última solicitud Beat.

El segundo nivel primero envía una solicitud de lectura a PutBuffer de SinkA y almacena los datos recibidos en la cola; si la tarea no necesita los datos o los datos se omiten de RefillBuffer, el nivel envía directamente una respuesta a través del canal D

El nivel 3 y el nivel 2 están conectados por una tubería, es decir, DataStorage ingresa al nivel 3 solo después de devolver los datos. Para solicitudes generales, esta etapa envía una respuesta con los datos leídos a través del canal D; para solicitudes Put, esta etapa envía una respuesta AccessAck al procesar el primer Beat.

El nivel 4 envía una solicitud de escritura a DataStorage



#### Comprobación de derivación

Para la misma solicitud, SourceD debe tener la prioridad más baja. La prioridad de cada canal en DataStorage se determina en realidad según la prioridad de la tarea de la misma solicitud. Sin embargo, dado que MSHR se libera antes de que se complete la solicitud a SourceD, se violarán las reglas de prioridad en DataStorage. En el caso de las solicitudes de adquisición, se liberan cuando se recibe el GrantAck, pero el GrantAck del cliente se puede otorgar cuando se recibe el Grant por primera vez. En este caso, MSHR lo libera cuando recibe el GrantAck antes del último Grant. Para las solicitudes Get, no se requiere GrantAck. Siempre que SourceD lo envíe, se liberará MSHR. En el caso de una versión anticipada, Get y Acquire pueden seguir leyendo datos de SourceD cuando llegue la siguiente solicitud para el mismo conjunto y emita una operación de escritura en el conjunto. En términos de corrección, la lectura anterior debe ser Debe realizarse Primero, pero SourceD tiene la prioridad más baja en DataStorage, por lo que la solución es extraer el Set/Way de SourceD y compararlo con el canal que se va a escribir. Si el Set/Way es el mismo, bloquéelos y deje que SourceD lo haga. primero.



## FuenteE

Recibir solicitud de MSHR y enviar GrantAck desde el canal E.
