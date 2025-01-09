# Integración SoC<br>Integración del sistema en chip

Esta sección describe los detalles funcionales de las interfaces de nivel superior de NANHU.
Vale la pena señalar que estas son las configuraciones típicas y el usuario puede especificar diferentes parámetros de arquitectura.<br>Este capítulo presenta los detalles funcionales de la interfaz de nivel superior de Southlake.
Vale la pena señalar que estas son configuraciones típicas y los usuarios pueden especificar diferentes parámetros de arquitectura.

## Reloj y reinicio Reloj y reinicio

NANHU tiene dos dominios de reloj:

* Un dominio de reloj para el núcleo de la CPU (`io_clock`). Todo el reloj del núcleo de la CPU es `io_clock`.<br>Dominio de reloj del núcleo del procesador (`io_clock`). Todos los relojes en el núcleo del procesador son `io_clock`.

* Un dominio de reloj para jtag (`io_systemjtag_jtag_TCK`).<br>Dominio de reloj JTAG (`io_systemjtag_jtag_TCK`).

NANHU tiene las siguientes señales de reinicio.
Todas las señales de reinicio son reinicios asincrónicos.<br>Southlake tiene las siguientes señales de reinicio.
Todas las señales de reinicio son reinicios asincrónicos.

* `io_reset` está sincronizado internamente con el reloj `io_clock`. Activo alto.d<br>`io_reset` está sincronizado internamente con el reloj `io_clock`. El nivel alto es válido.

* `io_systemjtag_reset` está sincronizado internamente con el reloj `io_systemjtag_jtag_TCK`. Activo alto.<br>`io_systemjtag_reset` está sincronizado internamente con el reloj `io_systemjtag_jtag_TCK`. El nivel alto es válido.

Hay dos tipos de sincronizadores en el RTL. Todas las instancias se utilizan para sincronizar el reinicio.<br>Hay dos tipos de sincronizadores en el RTL. Todas las instancias se utilizan para el reinicio sincrónico.

* `AsyncResetSynchronizerPrimitiveShiftReg` (sincroniza el reinicio de `jtag_clock` a `io_clock`, de baja frecuencia a alta frecuencia)

* `ResetGen/ResetGenDFT` (se utiliza para sincronizar tanto `jtag_clock` como `io_clock`, dependiendo de las instancias)

## Interfaz AXI Interfaz de bus AXI

Para nuestras definiciones de los tres puertos,<br>A continuación se muestran las definiciones de las tres interfaces de bus AXI en Southlake.

- Puerto de memoria AXI de 256 bits: para que la CPU acceda a DDR (la CPU es el maestro)

Espacio de direcciones: espacio DDR (0x8000_0000 a 0xf_ffff_ffff)

Ancho de ID AXI: 6 bits<br>Ancho de ID AXI: 6 bits

Número de transacciones pendientes: 56 (máximo admitido por caché L3)

Tamaño máximo de transacción: 64 bytes<br>Tamaño máximo de transacción: 64 bytes

Número de identificaciones: 64

Número máximo de transacciones pendientes por ID: 1
Número máximo de transacciones pendientes por ID: 1

- Puerto periférico AXI de 64 bits: para que la CPU acceda a los periféricos (la CPU es el maestro)

Espacio de direcciones: espacio de esclavos periféricos (0x0000_0000 a 0x7fff_ffff)

Ancho de ID AXI: 2 bits<br>Ancho de ID AXI: 2 bits

Número de pendientes: 1 para instrucción y 1 para datos para cada núcleo

Tamaño máximo de transacción: 8 bytes<br>Tamaño máximo de transacción: 8 bytes

Número de identificaciones: 4 (aunque la identificación es de 4 bits, la mayoría de ellas no se utilizan)

Número máximo de transacciones pendientes por ID: 1

- Puerto DMA AXI de 256 bits: para que los periféricos accedan a la CPU/DDR (la CPU es el esclavo)

Espacio de direcciones: espacio DDR (0x8000_0000 a 0xf_ffff_ffff)

Ancho de ID AXI: 8 bits<br>Ancho de ID AXI: 8 bits

Número de transacciones pendientes admitidas por la CPU: 32<br>Número de transacciones pendientes admitidas por la CPU: 32

Se admiten distintos tamaños de transacción y transacciones por ID. Se convierten en transacciones TileLink de 256 bits de forma interna. Una transacción AXI se puede dividir en varias transacciones TileLink.<br>La interfaz AXI admite distintos tamaños de transacción y transacciones por ID. Se convierten internamente en transacciones TileLink de 256 bits. Una transacción AXI se puede dividir en múltiples transacciones TileLink.

Este puerto DMA permite que los periféricos accedan a la memoria DDR en coherencia con las cachés de la CPU. Pueden enviar solicitudes AXI aw/w/ar estándar a este puerto para escribir o leer la memoria DDR. La coherencia de la memoria está respaldada en el puerto DMA por la caché L3. No se necesita ninguna acción explícita para sincronizar los datos con la CPU.<br>El puerto DMA permite que los periféricos accedan a DDR mientras el caché de la CPU es coherente. Pueden enviar solicitudes AXI aw/w/ar estándar a este puerto para escribir/leer DDR. La coherencia de la memoria en el puerto DMA está garantizada por la caché L3. No se requiere ninguna operación explícita para sincronizar datos con la CPU.

Nota: RISC-V define un orden de memoria débil. No es necesario que las instrucciones de carga y almacenamiento sean visibles para otros núcleos (núcleos de CPU, dispositivos DMA, etc.) antes de una instrucción de barrera de memoria explícita como "fence". Por ejemplo, en NANHU, Los datos en la cola de almacenamiento y el búfer de almacenamiento comprometido no son coherentes con el
Cachés de CPU. Para leer datos recién almacenados a través del puerto AXI de DMA, se debe utilizar una instrucción de bloqueo en el software para que los almacenamientos de CPU sean visibles para la jerarquía de caché coherente. Este es un requisito de RISC-V, no un error o problema de NANHU. Tenga en cuenta: RISC-V define un orden de memoria débil. Las instrucciones de carga y almacenamiento no necesitan ser visibles para otros subprocesos de hardware (núcleos de CPU, dispositivos DMA, etc.) antes de una instrucción de barrera de memoria explícita como fence. Por ejemplo, en Southlake, los datos en la cola de almacenamiento y en el búfer de almacenamiento de confirmación no son consistentes con la memoria caché de la CPU. Para leer datos recién almacenados a través del puerto DMA AXI, se debe utilizar una instrucción de barrera en el software para que los datos almacenados por la CPU sean visibles para la jerarquía de caché coherente. Este es un requisito de RISC-V, no un error/problema con Southlake.


## Mapa de memoria Mapa de memoria

Hay tres regiones principales:<br>El mapa de memoria tiene tres regiones principales:

* Espacio de esclavos periféricos 0x0000_0000 a 0x7fff_ffff<br>Espacio de dispositivo esclavo periférico (0x0000_0000 a 0x7fff_ffff)

* Espacio DDR: 0x8000_0000 a 0xf_ffff_ffff<br>Espacio DDR (0x8000_0000 a 0xf_ffff_ffff)

* DMA para acceder a DDR: 0x8000_0000 a 0xf_ffff_ffff<br>DMA para acceder a DDR (0x8000_0000 a 0xf_ffff_ffff)

Los periféricos en el espacio de direcciones de periféricos están determinados principalmente por el SoC.<br>Los periféricos en el espacio de direcciones de periféricos están determinados principalmente por el SoC.

El PMA predeterminado se configura según el mapa de memoria NANHU. El acceso ilegal a la memoria provocará excepciones de error de acceso.
Recuerde actualizar la configuración predeterminada de PMA si necesita cambiarla.
