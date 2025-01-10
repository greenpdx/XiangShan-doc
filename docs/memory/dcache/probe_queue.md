# Cola de sonda

La cola de sonda contiene 16 entradas de sonda, que son responsables de recibir solicitudes de sonda de la caché L2, convertir las solicitudes de sonda en señales internas y enviarlas a la tubería principal, donde se modifican los permisos de los bloques de sonda.

## Programación de solicitudes de sondeo

El flujo de procesamiento después de que la cola de sonda recibe la solicitud de sonda de L2 es el siguiente:

* Asignar una entrada de sonda vacía;
* Enviar una solicitud de sondeo a la tubería principal. Por cuestiones de tiempo, la solicitud se retrasará un segundo.
* Espere a que la tubería principal devuelva una respuesta;
* Liberar entrada de sonda.

## Reponer

### Procesamiento de alias

La arquitectura NanHu utiliza una caché VIPT de 128 KB, lo que genera el problema del alias de caché. La arquitectura NanHu resuelve el problema del alias desde una perspectiva de hardware. Como se muestra en la figura siguiente, el directorio de caché L2 mantiene el bit de alias correspondiente a cada bloque físico (que es decir, el índice de dirección virtual excede la parte de desplazamiento de página) garantiza que una determinada dirección física tenga solo un bit de alias en la caché L1. Cuando la caché L1 desea obtener un bit de alias diferente, es decir, un índice virtual diferente, en una cierta dirección física, la caché L2 sondeará el bloque de caché correspondiente a otro índice virtual.

![dcache-probe-queue-alias.jpg](../../figs/memblock/dcache-probe-queue-alias.jpg)

Dado que las solicitudes de sondeo enviadas por L2 se basan todas en direcciones físicas, pero L1 accederá a la memoria caché en modo VIPT, L1 también necesita conocer el bit de alias del bloque sondeado. La arquitectura NanHu utiliza el campo de datos del canal B en el Protocolo TileLink para transmitir 2 bits Después de recibir la solicitud, la cola de sonda combinará el bit de alias y el desplazamiento de página para obtener el índice necesario para acceder a DCache.

### Soporte de instrucciones atómicas

En el diseño de XiangShan, las operaciones atómicas (incluida lr-sc) se realizan en DCache. Al ejecutar la instrucción LR, se garantiza que la dirección de destino ya esté en DCache. Para simplificar el diseño, LR registrará un conjunto de reservas en la tubería principal, registra la dirección de LR y bloquea la sonda en esa dirección. Para evitar un bloqueo, la tubería principal dejará de bloquear la sonda después de esperar a SC durante un cierto período de tiempo (determinado por los parámetros `LRSCCycles` y `LRSCBackOff`). ). Cualquier comando SC recibido se considera como un error de SC. Por lo tanto, después de que LR registra el conjunto de reservas y espera el emparejamiento de SC, es necesario bloquear la solicitud de sonda para operar en DCache.
