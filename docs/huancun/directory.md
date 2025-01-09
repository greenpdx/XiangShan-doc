# Diseño de directorio

Este capítulo presentará el diseño del directorio en huancun. El “catálogo” al que se hace referencia en este capítulo es amplio e incluye metadatos y etiquetas.

huancun es un caché no inclusivo basado en una estructura de directorio, y su diseño se inspiró en NCID[^ncid]. Entre los cachés multinivel, los datos no son inclusivos, mientras que los directorios sí lo son. En términos de organización estructural, huancun almacena los directorios de datos de nivel superior y datos locales por separado, y las estructuras de ambos son similares. El directorio se construye utilizando SRAM, con Set como índice, y cada Set contiene datos Way.



Cada elemento del directorio de datos local almacena la siguiente información:

* estado: el permiso para guardar el bloque de datos (uno de los siguientes: Punta/Tronco/Rama/Inválido)
* sucio: indica si el bloque de datos está sucio
* clientStates: Guarda los permisos del bloque de datos en la capa superior (solo tiene sentido cuando el bloque no es inválido)
* prefetch: indica si el bloque de datos está precargado



Cada elemento del directorio de datos superior almacena la siguiente información:

* estado: el permiso para guardar el bloque de datos superior
* alias: guarda el último bit de la dirección virtual del bloque de datos de la capa superior (es decir, el bit de alias, consulte [Problema de alias de caché](./cache_alias.md) para obtener más detalles)



## Lectura de directorio

Cuando el módulo Alloc de MSHR asigna una solicitud al MSHR, también inicia simultáneamente una solicitud de lectura al directorio. Lea la capa superior y los metadatos y etiquetas locales en paralelo, determine si hay un resultado según la comparación de etiquetas y pase los metadatos de la ruta correspondiente al MSHR correspondiente según la situación del resultado.

Cuando el directorio impacta, la ruta pasada es la ruta impactada; cuando el directorio no impacta, la ruta pasada es la ruta obtenida según el algoritmo de reemplazo. El algoritmo de reemplazo se puede configurar de forma flexible. El directorio de datos local de la versión Nanhu utiliza el algoritmo de reemplazo PLRU y el directorio de datos de nivel superior utiliza el algoritmo de reemplazo aleatorio.



## Escritura en directorio

Cuando la transacción MHSR está llegando a su fin, generalmente es necesario escribir el directorio para actualizar el estado, las etiquetas y otra información. El directorio tiene 4 puertos de solicitud de escritura, que reciben la escritura de metadatos locales, metadatos de nivel superior, etiquetas locales y etiquetas de nivel superior respectivamente. Las solicitudes de escritura tienen mayor prioridad que las solicitudes de lectura.

En términos de relación de conexión, los 4 tipos de solicitudes de escritura anteriores de todos los MSHR se arbitran por separado en el directorio. La relación de prioridad de arbitraje está cuidadosamente diseñada para evitar errores de anidación de solicitudes o bloqueos.



## Problemas comunes y consideraciones de diseño

#### Ya hay metadatos de nivel superior en el directorio, ¿por qué clientStates todavía se almacena en los metadatos locales?

* Cuando una solicitud, como Adquirir BLOCK_A, falla en el directorio local, el directorio seleccionará una ruta de información para pasar a MSHR según el algoritmo de reemplazo. En este momento, es posible que esta ruta no sea inválida, pero tiene un bloque de datos BLOQUE_B. Para completar esta solicitud de adquisición, necesitamos conocer la información de estado de BLOCK_B en la capa superior. Para evitar leer el directorio dos veces, almacenamos una copia adicional de clientStates en los metadatos locales.



#### ¿Existe riesgo de contención de lectura y escritura en el directorio?

* Nuestro MSHR está bloqueado según Set, y se permite que nuevas solicitudes ingresen y lean el directorio solo después de que se libera MSHR, por lo que no hay riesgo de contención de lectura y escritura.



[^ncid]: Zhao, Li, et al. "NCID: una caché no inclusiva, una arquitectura de directorio inclusiva para jerarquías de caché flexibles y eficientes". *Actas de la 7.ª conferencia internacional de la ACM sobre fronteras informáticas*. 2010.
