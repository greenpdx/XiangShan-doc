# Tubería de almacenamiento

Este capítulo presenta el diseño de la tubería de dirección de almacenamiento/datos de almacenamiento del procesador Xiangshan (arquitectura Nanhu) y el flujo de procesamiento de la instrucción de almacenamiento.

El procesador Xiangshan (arquitectura Nanhu) utiliza un modo de ejecución independiente para la dirección de almacenamiento y los datos. La dirección de almacenamiento y los datos se pueden enviar desde la estación de reserva cuando estén listos e ingresar al flujo de procesamiento posterior. Los datos/direcciones correspondientes a la misma instrucción de almacenamiento Las dos operaciones están vinculadas entre sí por el mismo robIdx / sqIdx.

El procesador Xiangshan (arquitectura Nanhu) contiene dos canales de direcciones de almacenamiento (sta). Cada canal de direcciones de almacenamiento se divide en cuatro etapas de canalización. Las dos primeras etapas de canalización son responsables de pasar la información de control de almacenamiento y la dirección a la cola de almacenamiento, y la última etapa de canalización es responsable de enviar la información de control de almacenamiento y la dirección a la cola de almacenamiento. Dos etapas de la canalización son responsables de pasar la información de control de la tienda y la dirección a la cola de la tienda. El nivel es responsable de esperar a que se complete la comprobación de dependencia del acceso a la memoria.

El procesador Xiangshan (arquitectura Nanhu) contiene dos canales de datos de la tienda (estándar). Una vez que la estación de reserva proporciona los datos de la tienda, el canal de datos de la tienda los escribe inmediatamente en el elemento correspondiente de la cola de la tienda.

## Tubería Sta

La división de cada nivel del pipeline es la siguiente:

![tubería de almacenamiento](../../figs/memblock/tubería-de-almacenamiento.png)

Etapa 0

* Calcular dirección virtual
* La dirección virtual se envía a TLB

Etapa 1

* TLB genera dirección física
* Complete una comprobación rápida de anomalías
* Iniciar la comprobación de dependencia del acceso a la memoria
*La dirección física se envía a la cola de la tienda.

Etapa 2

* Comprobación de dependencia de acceso a memoria
* Complete todas las comprobaciones de excepciones y consultas de PMA, y actualice la cola de la tienda en función de los resultados.

Etapa 3

* Comprobación completa de la dependencia de acceso a la memoria
* Notificar a ROB que puede enviar instrucciones

El capítulo [Cola de almacenamiento](../lsq/store_queue.md) describe los detalles del proceso de actualización de la cola de almacenamiento.

## Tubería estándar

Etapa 0

* La estación reservada proporciona datos de la tienda.
* Escribe datos de la tienda en la cola de tiendas

## Detalles adicionales sobre la implementación de la tienda

**Manejo de errores de TLB.** Al igual que el pipeline de carga, el pipeline de direcciones de almacenamiento también puede experimentar errores de TLB. El manejo de ambos pipelines es básicamente el mismo, consulte [Manejo de errores de TLB de carga](./load_pipeline.md#tlb -miss) . store addr utiliza solo un puerto rsFeedback para proporcionar retroalimentación a la estación de reserva si la operación de cálculo de store addr necesita ser reenviada desde la estación de reserva.

**PMA y verificación de excepciones.** Por cuestiones de tiempo, MMIO y otros resultados de verificación no se completan hasta que los datos se actualizan en la cola de almacenamiento durante un ciclo. En este momento, la canalización de almacenamiento escribirá el resultado final de la consulta en la cola de la tienda.
