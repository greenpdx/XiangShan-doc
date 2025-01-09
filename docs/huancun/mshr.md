Diseño #MSHR

Este capítulo presenta el diseño de MSHR (Miss Status Holding Registers) no inclusivo en huancun.

Cuando Huancun recibe una solicitud de adquisición/liberación de un caché superior o una solicitud de sondeo de un caché inferior, asigna un MSHR para la solicitud y obtiene la información de permisos de la dirección en el caché en este nivel y superiores leyendo el directorio. Con base en esta información, decida:

* [Control de permisos](#meta_control): Cómo actualizar los permisos en el directorio propio/directorio del cliente;

* [Control de solicitud](#request_control): si se deben enviar subsolicitudes a los cachés superior e inferior y esperar respuestas a estas subsolicitudes;
<!--
HACER
* Cómo manejar solicitudes anidadas, etc.
-->
A continuación se presentará el diseño de huancun MSHR desde estos aspectos utilizando L2 Cache como ejemplo.


<h2 id=meta_control>Control de permisos</h2>

Los datos en huancun se almacenan de forma no inclusiva, pero el directorio adopta estrictamente la estrategia de inclusión, por lo que MSHR puede obtener toda la información de permisos de la dirección solicitada en ICache, DCache y L2 Cache, controlando así el cambio del permiso de la dirección. Cada dirección del sistema de caché de XiangShan sigue las reglas del árbol de consistencia de TileLink. Cada bloque tiene cuatro estados en cada capa del sistema de caché: N (Ninguno), B (Rama), T (Tronco) y TT (TIP). Los primeros tres corresponden a sin permiso, solo lectura y legible y escribible. El árbol de consistencia crece de abajo hacia arriba en el orden de la memoria, L3, L2 y L1. La memoria, como nodo raíz, tiene permisos de lectura y escritura. En cada capa, los permisos de los nodos secundarios no pueden superar los permisos de los nodos secundarios. el nodo padre. Entre ellos, TT representa un nodo hoja en una rama con permiso T, lo que significa que la capa superior de este nodo solo tiene permiso N o B. Por el contrario, un nodo con permiso T pero sin permiso TT significa que debe haber un Nodo de permiso T/TT en la capa superior. Para conocer las reglas detalladas, consulte el Capítulo 9.1 del Manual de TileLink.

MSHR actualiza el bit sucio, el campo de permiso y el campo clientStates del directorio propio según el tipo de solicitud y el resultado de la lectura del directorio. ClientStates indica que si la dirección tiene permiso (B o superior) en la memoria caché actual, entonces esta dirección está en la caché L1 superior. Además, MSHR también actualizará el directorio de clientes correspondiente a ICache y DCache, incluido el dominio de permisos y el dominio de alias (que se presentarán en el capítulo [Problema de alias de caché](./cache_alias.md)) ).

Los siguientes dos ejemplos ilustran brevemente cómo MSHR determina los cambios de permisos y por qué estos campos son necesarios en el diseño de directorios.

1. Si DCache tiene permiso para direccionar X, pero ICache no, ICache solicita permiso para X desde L2. En este momento, L2 debe determinar si L2 tiene el bloque X, si DCache tiene el bloque X y, de ser así, si los permisos son suficientes. En función de esta información, MSHR decidirá si devolver los datos directamente a ICache o solicitar a DCache datos (Sonda) y luego devolverlo a ICache, aún necesita adquirir este bloque de L3 y luego reenviarlo a ICache, por lo que necesitamos mantener los bits de permiso en el directorio propio y en el directorio del cliente respectivamente.

2. Si DCache tiene permiso para la dirección X, pero L2 no, entonces DCache reemplaza X y envía una solicitud para liberar la dirección X. Después de recibir la solicitud, L2 asigna un MSHR. Al mismo tiempo, esperamos que L2 pueda guardar DCache Si es necesario reemplazar el bloque X liberado, L2 también debe liberar el bloque de reemplazo Y a L3 de acuerdo con la estrategia de reemplazo. En este ejemplo, necesitamos saber si Y son datos sucios en L2, lo que implica si se deben traer datos al liberar Y a L3, por lo que el bit sucio es necesario en el directorio propio; al reemplazar Y fuera de L2, necesitamos saber ¿Cuál debería ser la versión? ¿Qué tipo de parámetro se incluye? Por lo tanto, el dominio clientStates debe mantenerse en el directorio propio para determinar la autoridad del nodo superior en la dirección Y.

Para obtener más detalles sobre el directorio, lea [Diseño de directorio](./directory.md).


<h2 id=request_control>Control de solicitud</h2>

MSHR debe determinar qué subsolicitudes deben completarse en función del contenido de la solicitud y el resultado de la lectura del directorio, incluyendo si se debe adquirir o liberar hacia abajo, si se debe sondear hacia arriba, si se debe activar una búsqueda previa, si se debe modificar el directorio y etiqueta, etc. Solicitud, MSHR también necesita registrar qué subsolicitudes deben esperar respuestas.

MSHR especifica que estas solicitudes se programen y las respuestas que se esperen como eventos, y utiliza una serie de registros de estado para registrar si estos eventos se completan. Los registros `s_*` indican la solicitud que se debe programar y los registros `w_*` indican la respuesta que se debe esperar. Después de que MSHR obtenga el resultado de la lectura del directorio, establecerá los eventos que deben completarse (` Los registros s_*` y `w_*` se configuran como `false.B`, lo que indica que no se ha enviado la solicitud o que no se ha recibido la respuesta. Una vez completado el evento, el registro se configura como `true.B` Cuando se completen todos los eventos, se publicará el MSHR.
