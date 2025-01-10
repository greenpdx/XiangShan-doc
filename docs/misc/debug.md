Uso de GDB para depurar Xiangshan
===================

Hemos conectado el módulo de depuración de Rocket Chip a Xiangshan para admitir la versión 0.13 [Manual de depuración de RISCV](https://riscv.org/wp-content/uploads/2019/03/riscv-debug-release.pdf) Modo de depuración en .

## Principio

El módulo de depuración puede poner el núcleo del procesador en modo de depuración a través de un método de interrupción, y luego GDB puede hacer que el núcleo del procesador ejecute cualquier instrucción y lea cualquier registro. Después de que el módulo de depuración emite una interrupción, el núcleo del procesador caerá en el espacio de direcciones MMIO del módulo de depuración y comenzará un bucle. El módulo de depuración controla el comportamiento del núcleo y lee los registros del núcleo escribiendo instrucciones en este espacio de direcciones.

El módulo de depuración también admite la función de acceso al bus del sistema, lo que significa leer y escribir memoria física sin pasar por el núcleo del procesador. El módulo de depuración de Xiangshan está conectado a la caché L3 a través de TileLink para facilitar el mantenimiento de la consistencia de los datos. Tenga en cuenta que para utilizar esta función en la simulación, utilice la configuración predeterminada.

## Software y configuración
Para depurar Xiangshan con GDB, necesita instalar `riscv64-unknown-elf-gdb` y `openocd`. Para obtener instrucciones de instalación, consulte los proyectos de Github correspondientes (siga directamente el README para instalar, no se requiere configuración adicional): [GDB](https://github.com/riscv-collab/riscv-binutils-gdb), [ OpenOCD](https://github.com/riscv/riscv-openocd)

OpenOCD requiere un archivo de configuración, puedes utilizar la siguiente configuración:
```
adapter speed 10000

adapter driver remote_bitbang
remote_bitbang host localhost
remote_bitbang port 23334

set _CHIPNAME riscv
jtag newtap $_CHIPNAME cpu -irlen 5

set _TARGETNAME $_CHIPNAME.cpu
target create $_TARGETNAME riscv -chain-position $_TARGETNAME 

# riscv set_reset_timeout_sec 120
# riscv set_command_timeout_sec 30
riscv set_mem_access sysbus
init
halt
echo "Ready for Remote Connections"
```
## Conéctese a Xiangshan mediante GDB
Después de compilar emu, agregue los dos indicadores `--enable-jtag --no-diff` al ejecutar, y ejecute `openocd -f config.cfg` en otra terminal, donde `config.cfg` es el archivo de configuración de OpenOCD, y ejecute `riscv64-unknown-elf-gdb` en la tercera terminal. Se recomienda utilizar la pantalla dividida tmux para la operación.

Ejecute `target extended-remote localhost:3333` en la línea de comando GDB y espere un momento para conectarse al emulador Xiangshan.
