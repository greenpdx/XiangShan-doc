# JTAG y depuración

NANHU implementa [RISC-V External Debug Support versión 0.13.2](https://riscv.org/wp-content/uploads/2019/03/riscv-debug-release.pdf). -V soporte de depuración externa versión 0.13.2. 2](https://riscv.org/wp-content/uploads/2019/03/riscv-debug-release.pdf).

NANHU utiliza componentes que incluyen el módulo de depuración, la interfaz del módulo de depuración, el módulo de transporte de depuración y la interfaz JTAG.

El módulo de depuración NANHU admite el búfer de programa (16 bytes) y el acceso al bus del sistema. Los comandos abstractos se implementan mediante el búfer de programa.
Está conectado a la barra transversal L3 mediante TileLink para admitir el acceso al bus del sistema.
Tiene dos partes: DMInner y DMOuter. DMInner es controlado por el reloj del núcleo, mientras que DMOuter es controlado por el reloj JTAG. DMOuter emite interrupciones de depuración.<br>El módulo de depuración de Nanhu admite el acceso al búfer de programa (16 bytes) y al bus del sistema. Los comandos abstractos se implementan a través de buffers de programa.
Se conecta a la interconexión L3 a través de TileLink para admitir el acceso al bus del sistema.
Consta de dos partes: DMInner y DMOuter. DMInner es impulsado por el reloj central, mientras que DMOuter es impulsado por el reloj JTAG. DMOuter activa una interrupción de depuración.

## Depuración

NANHU CSR tiene un registro de un bit que indica si está en modo de depuración.

Se puede ingresar al modo de depuración cuando:

* El módulo de depuración emite una interrupción de depuración.<br>El módulo de depuración activó una interrupción de depuración.

* Se ejecuta un ebreak en modo X y se ejecuta ebreakX (X es M, S o U).

* El hart ha regresado del modo de depuración y el bit de paso en dcsr está configurado. El hart ingresa al modo de depuración después de que se haya confirmado exactamente una instrucción. Configurar. Cuando se ejecuta una instrucción, el hilo de hardware entra en modo de depuración.

* El disparador se activa con el bit de acción establecido en 1 en el CSR de mcontrol correspondiente.<br>Cuando se activa el disparador, el bit de acción correspondiente en el CSR de mcontrol se establece en 1

NANHU implementa los siguientes registros de modo de depuración:
Registro de estado y control de depuración (dcsr), Debug PC (dpc), Debug Scratch (dscratch y dscratch1).<br>Southlake implementa los siguientes registros de modo de depuración:
Registro de estado y control de depuración (dssr), PC de depuración (dpc), registros de bloc de notas de depuración (dscratch y dscratch1).

### Control de depuración y registro de estado Control de depuración y registro de estado

NANHU implementa bits dcsr opcionales: ebreaks, ebreaku, mprven y step.
Consulte [RISC-V External Debug Support Version 0.13.2](https://riscv.org/wp-content/uploads/2019/03/riscv-debug-release.pdf) para obtener más detalles.<br>NANHU ha implementado Los bits dcsr opcionales son ebreaks, ebreaku, mprven y step.
Consulte [RISC-V External Debug Support Release 0.13.2](https://riscv.org/wp-content/uploads/2019/03/riscv-debug-release.pdf) para obtener más detalles.

### Depuración de PC Depuración de PC

El módulo de depuración de la PC almacena el contador del programa de la instrucción exacta que entró en modo de depuración. El PC se recupera cuando se ejecuta un dret. El módulo de depuración puede modificar este CSR para manipular la ruta de ejecución del programa. modo al contador del programa para la instrucción exacta. La PC se restaura cuando se ejecuta la instrucción dret. El módulo de depuración puede modificar este CSR para controlar la ruta de ejecución del programa.

### Depuración Scratch Bloc de notas de depuración

Hay 2 registros de depuración (dscratch y dscratch1) en NANHU. Estos registros son utilizados como registros de depuración por el módulo de depuración.<br>Hay 2 registros de depuración (dscratch y dscratch1) en NANHU. El módulo de depuración utiliza estos registros como registros de borrador.

## Desencadenar

### Registro de selección de disparador Registro de selección de disparador

Tselect selecciona uno de los 10 activadores de NANHU. Todos los activadores de NANHU se enumeran a continuación.<br>Tselect selecciona uno de los 10 activadores de NANHU. La siguiente tabla enumera todos los desencadenantes de Southlake.

| No. | Tipo | Broca de sincronización | Broca de cadena | Broca de selección | Broca de coincidencia |
| --- | --- | --- | --- | --- | --- |
| 0 | ejecutar/almacenar/cargar | Conectado a 0 | Y | Y* | 0, 2 o 3 |
| 1 | ejecutar/almacenar/cargar | | Y | Y* |
| 2 | ejecutar/almacenar/cargar | | Y | Y* |
| 3 | ejecutar/almacenar/cargar | | Y | Y* |
| 4 | ejecutar/almacenar/cargar | | Y | Y* |
| 5 | ejecutar/almacenar/cargar | | Y | Y* |
| 6 | ejecutar/almacenar/cargar | | Y | Y* |
| 7 | ejecutar/almacenar/cargar | | Y | Y* |
| 8 | ejecutar/almacenar/cargar | | Y | Y* |
| 9 | ejecutar/almacenar/cargar | | Conectado a 0 | Y* |

*: Si el tipo es almacenar o cargar, el bit de selección será WARL como 0.<br>Si el tipo es almacenar o cargar, el bit de selección se establecerá en WARL como 0.

### Registro de estado de datos de activación Registro de estado de datos de activación

Todos los activadores de NANHU son activadores de control de coincidencia. Cada uno tiene un registro mcontrol (tdata1), un registro de datos (tdata2) y un registro de información (tinfo).<br>Todos los activadores de NANHU son activadores de control de coincidencia. Cada flip-flop tiene un registro mcontrol (tdata1), un registro de datos (tdata2) y un registro de información (tinfo).

## Punto de interrupción

NANHU admite el uso de la instrucción ebreak como punto de interrupción de software. Para utilizarla en el modo privilegiado X, se debe configurar el bit ebreakX en dcsr. Los puntos de interrupción de hardware se implementan mediante activadores. Para utilizar esta instrucción en el modo privilegiado X, se debe configurar el bit ebreakX en dcsr. Los puntos de interrupción de hardware se implementan mediante activadores.

## Mapa de memoria de depuración Mapa de memoria de depuración

El módulo de depuración utiliza el espacio de direcciones (0x3802_0000, 0x3802_0fff).<br>El módulo de depuración utiliza el espacio de direcciones de 0x3802_0000 a 0x3802_0fff.

## Interfaz del módulo de depuración

DMI tiene un ancho de bus de 7 bits.<br>DMI tiene un ancho de bus de 7 bits.

## Interfaz JTAG Interfaz JTAG

La interfaz JTAG de Nanhu admite el reinicio asincrónico TRSTn.<br>La interfaz JTAG de Nanhu admite el reinicio asincrónico TRSTn.

## Conexión al módulo de depuración

Primero, es necesario compilar [riscv-openocd](https://github.com/riscv/riscv-openocd). Consulta el archivo README de Github](https://github.com/riscv/riscv-openocd/blob/riscv /README) sobre cómo compilar riscv-openocd.<br>Primero, debes compilar [riscv-openocd](https://github.com/riscv/riscv-openocd). Para obtener información sobre cómo compilar riscv-openocd, consulte el [README de Github](https://github.com/riscv/riscv-openocd/blob/riscv/README).

Ejecute simv con las opciones: `./difftest/simv [+workload=WorkloadName.bin] +no-diff
