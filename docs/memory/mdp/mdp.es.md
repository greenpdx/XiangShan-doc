# Predicción de dependencia de acceso a memoria

Este capítulo describe la arquitectura general del predictor de predicción de violaciones de acceso a memoria del procesador Xiangshan.

El procesador Xiangshan predice las dependencias de acceso a la memoria en función de la decodificación cercana a la PC. El código actual admite dos métodos de predicción:

* Cargar tabla de espera[^21264]
* Variantes de conjuntos de tiendas[^storesets]

La arquitectura de Southlake utiliza un mecanismo de predicción de dependencia de memoria similar a Store Sets. Si se predice que es probable que una instrucción de carga viole la excepción, la instrucción de carga debe esperar en la estación de reserva hasta que se emitan todos los almacenamientos anteriores antes de poder emitirse.

## Predictor de violación de acceso a memoria de Southlake

### Predicción de dependencia

* La idea general se basa en Store Sets, y se implementan dos tablas en Store Sets: SSIT y LFST
* La arquitectura de Southlake solo predice violaciones causadas por una dirección de tienda que no está lista
* Un conjunto de almacenamiento rastreará varios almacenamientos en curso al mismo tiempo. Solo cuando se ejecuten todos los almacenamientos aquí, se predecirá que la carga que depende de este conjunto de almacenamiento no tiene dependencia
* Las consultas SSIT se realizan en el cambio de nombre. Las consultas LFST se realizan en el envío.

### Manejo de dependencias

* El predictor de violación de acceso a memoria de la arquitectura Southlake utiliza robIdx para rastrear el contexto de las instrucciones.
* A diferencia de los conjuntos de tiendas originales, **no es necesario ejecutar secuencialmente las instrucciones de tienda dentro del mismo conjunto de tiendas**
* El predictor **intentará predecir si la instrucción de carga puede depender de los resultados de múltiples almacenamientos**. Si el predictor piensa que la instrucción de carga no depende de los resultados de múltiples almacenamientos, entonces después de la dirección del último dependiente Almacenar antes de que se genere esta carga. Se puede emitir esta instrucción de carga. De lo contrario, esta carga no se emitirá hasta que todos los almacenes de los que se prevé que dependa hayan generado direcciones.

### Actualización del predictor

Cuando se produce una violación de almacenamiento, el PC de la instrucción que desencadenó la violación se pasa al predictor de violación de acceso a memoria para su actualización. La información en el predictor de violación se invalida después de cada intervalo de actualización. El intervalo de actualización se puede configurar utilizando `slvpredctl " Configuración del registro CSR.

<!-- Cuando se actualiza el predictor de infracciones, el elemento correspondiente se invalida directamente. Más adelante, consideraremos agregar un diseño de nivel de confianza para eliminar solo los elementos con baja confianza. -->

<!-- Actualmente, el resultado de la carga normal no se enviará al predictor para su actualización. El diseño correspondiente se considerará en la próxima versión. -->

## Referencias

[^21264]: Kessler, Richard E. "El microprocesador alfa 21264". IEEE micro 19.2 (1999): 24-36.

[^storesets]: Chrysos, George Z. y Joel S. Emer. "Predicción de dependencia de memoria mediante conjuntos de almacenamiento". ACM SIGARCH Computer Architecture News 26.3 (1998): 142-153.
