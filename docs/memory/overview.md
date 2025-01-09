# Arquitectura general

El «bloque de memoria» del procesador Xiangshan incluye el canal de acceso a la memoria y la cola en el núcleo, así como la caché de datos de primer nivel estrechamente acoplada con el canal de acceso a la memoria. Usamos el término «unidad de acceso a la memoria» para referirnos a esta parte. del 'Memblock'.

![Canalización general](../figs/memblock/nanhu-memblock.png)

La unidad de acceso a la memoria en el núcleo del procesador Xiangshan (arquitectura Nanhu) se muestra en la figura anterior. Contiene dos [canalizaciones de carga](./fu/load_pipeline.md), dos [canalizaciones de almacenamiento](./fu/store_pipeline) independientes. md#Sta-Pipeline) y dos [tuberías estándar](./fu/store_pipeline.md#Std-Pipeline). [cola de carga](./lsq/load_queue.md) y [cola de almacenamiento](./ lsq/store_queue .md) es responsable de mantener la información de orden de las instrucciones de acceso a la memoria. La cola de carga será responsable de monitorear los resultados de recarga subsiguientes y realizar operaciones de escritura diferida cuando la instrucción de carga no esté en la caché de primer nivel. La cola de almacenamiento es responsable de almacenar temporalmente los datos antes de que se envíe la instrucción. almacenar datos y proporcionar datos para que la tienda los reenvíe a la carga.

Una vez que se confirma el comando de almacenamiento, la cola de almacenamiento moverá los datos que contiene al búfer de almacenamiento confirmado. El búfer de almacenamiento confirmado fusionará las solicitudes de escritura de almacenamiento en las líneas de caché y se utilizará cuando esté cerca de llenarse. Los múltiples almacenes fusionados Las solicitudes de escritura se escriben en el caché de datos de primer nivel al mismo tiempo.

El [caché de datos de nivel 1](../memory/dcache/dcache.md) expone dos puertos de lectura de 64 bits y un puerto de escritura con el mismo ancho que la línea de caché de datos de nivel 1 al componente de acceso a la memoria interna. así como un puerto de recarga de datos. El ancho del puerto de recarga de datos está determinado por el ancho del bus entre la caché de datos y la caché L2. Actualmente, el ancho del puerto de recarga de datos de la arquitectura Southlake es de 256 bits. Los datos de primer nivel La memoria caché utiliza el protocolo de bus TileLink.

[MMU](./mmu/mmu.md), también conocida como Unidad de gestión de memoria, es principalmente responsable de traducir direcciones virtuales en direcciones físicas en el procesador y luego utilizar esta dirección física para acceder a la memoria. Al mismo tiempo, también se realizan comprobaciones de permisos, como por ejemplo si se puede escribir o ejecutar. La [MMU](./mmu/mmu.md) de Xiangshan incluye [TLB](./mmu/tlb.md), [L2TLB](./mmu/tlb.md) (./mmu/l2tlb.md), [Repetidor](./mmu/mmu.md#repeater), [PMP y PMA](./mmu/pmp_pma.md) y otros componentes.

Varios componentes trabajan juntos para implementar el mecanismo de acceso a la memoria fuera de orden del procesador Xiangshan. Consulte la sección [Introducción al mecanismo de acceso a la memoria](./mechanism.md). Además, la sección [Predicción de violación de acceso a la memoria](./ mdp /mdp.md) también se describe aquí.

!!! nota
 La entrega real del código Verilog back-end implica el reemplazo de algunos códigos de ruta crítica.

El último diseño de la unidad de acceso a la memoria puede consultar este informe: [Introducción a la microestructura de la unidad de acceso a la memoria de Xiangshan (inglés)](https://github.com/OpenXiangShan/XiangShan-doc/blob/main/slides/20230426-LSU_of_Xiangshan_Processor.pdf ).
