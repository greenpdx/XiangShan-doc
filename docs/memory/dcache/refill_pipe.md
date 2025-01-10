# Tubo de recarga

En comparación con la arquitectura YanqiHu, la arquitectura NanHu agrega una tubería de relleno dedicada, Refill Pipe. Para una solicitud de falla de carga o almacenamiento, se seleccionará una ruta de reemplazo antes de ingresar a la cola de fallas. Después de obtener los datos devueltos por L2, se pueden enviar a el conducto de recarga. Por lo tanto, solo se necesita una operación para escribir los datos de recarga en DCache, sin tener que acceder a DCache nuevamente.

## Conflicto de lectura y escritura entre la tubería de recarga y la tubería principal

Tanto la tubería de recarga como la tubería principal escribirán en DCache. La tubería de recarga se completa de una sola vez, mientras que la tubería principal debe pasar por cuatro etapas de tuberías, como la lectura de etiquetas y la lectura de datos, antes de escribir datos en DCache. . Para garantizar la coherencia de los datos de lectura y escritura del front-end y back-end en la tubería principal, es decir, la lectura y escritura no se interrumpirán por la operación de escritura de la tubería de recarga. La tubería de recarga se bloqueará y se almacenará temporalmente. No se puede escribir en DCache en los siguientes casos:

* Existe un conflicto de conjunto entre la solicitud de la tubería de recarga y la etapa 1 de la tubería principal;
* La solicitud de Refill Pipe ha establecido un conflicto con la Etapa 2 / Etapa 3 de Main Pipe y tiene la misma forma de habilitar la señal `way_en`.

La tubería principal completará la comparación de etiquetas en la Etapa 1 y obtendrá `way_en`, pero debido al cronograma ajustado, la estrategia de bloqueo de la Etapa 1 de la tubería principal para la tubería de recarga se relaja aquí, y solo se realiza el bloqueo de acuerdo con el conjunto. suficiente.
