# Diseño de la arquitectura general del procesador Xiangshan

El procesador Xiangshan está diseñado con una estructura de seis números fuera de orden y actualmente admite la extensión RV64GCBK (la cadena de instrucciones específica es `RV64IMAFDC_zba_zbb_zbc_zbs_zbkb_zbkc_zbkx_zknd_zkne_zknh_zksed_zksh_svinval_zicbom_zicboz`). La tubería frontal del procesador Xiangshan incluye una unidad de predicción de bifurcaciones, una unidad de búsqueda de instrucciones, un búfer de instrucciones y otras unidades que buscan instrucciones de forma secuencial. El back-end incluye decodificación, cambio de nombre, reordenamiento de buffer, estación de reserva, archivo de registro de punto flotante/entero y unidad de operación de punto flotante/entero. Separamos el subsistema de acceso a la memoria, incluyendo dos pipelines de carga, dos pipelines de direcciones de almacenamiento, dos pipelines de datos de almacenamiento, así como colas de carga y colas de almacenamiento independientes, buffers de almacenamiento, etc. El caché incluye módulos como ICache, DCache, L2/L3 Cache (HuanCun), TLB y prefetcher. La posición de cada parte en la tubería y la configuración de los parámetros se pueden obtener de la siguiente figura.

![Diagrama de la arquitectura de Xiangshan](./figs/nanhu.png)

Para un diseño estructural detallado, consulte la introducción del capítulo correspondiente.
