# Prototipo FPGA

Guía de construcción de sistemas mínimos de código abierto FPGA basada en Nanhu

## Pasos básicos

### Compilar Xiangshan Nanhu RTL

(1) La rama principal (arquitectura Kunming Lake) se encuentra en desarrollo continuo. La versión de salida de cinta de la arquitectura Nanhu está diseñada en la rama Nanhu, y su rama adaptada al entorno FPGA se encuentra en [nanhu-clockdiv2](https: //github.com/OpenXiangShan/ XiangShan/árbol/nanhu-clockdiv2)

Aviso:

* Debido a que el código FPGA y el código de simulación no son el mismo conjunto, si usa el mismo repositorio XiangShan, necesita `rm -rf build` para eliminar los archivos de salida existentes
* El código Nanhu de doble núcleo se ha adaptado de acuerdo con la capacidad de recursos del FPGA. Actualmente, el código de doble núcleo se ha enviado para la verificación del FPGA con [09d2a32142c64fca898e17c0b943e61ddc331958](https://github.com/OpenXiangShan/XiangShan/commit/09d2a32142c64fca898e17c0b943e61ddc331958).



(2) Extraiga el repositorio de GitHub

``mierda
clon de git https://github.com/OpenXiangShan/XiangShan
disco compacto Xiang Shan
exportar NOOP_HOME=$(pwd)
verificación git 09d2a32142c64fca898e17c0b943e61ddc331958
hacer inicialización
hacer limpio
hacer verilog NUM_CORES=2 # XiangShan de doble núcleo
```

(3) Realice algunas modificaciones al código generado para que Vivado pueda llamar a los recursos de ultra RAM durante la instanciación. Por ejemplo, en un archivo que comienza con array_, declare una RAM de 64 bits con una profundidad mayor a 1024, lo que obliga a Vivado a usar RAM ultra para la instanciación, lo que reduce el uso de bram.

Modificar `array_16_ext.v`:

![matriz_16_ext.png](../../figs/fpga_images/matriz_16_ext.png)


### Copiar scripts relacionados con Vivado, generar proyecto Vivado, compilar flujo binario

``mierda
# preparar los guiones
clon de git https://github.com/OpenXiangShan/env-scripts.git
CD xs_nanhu_fpga
hacer update_core_flist CORE_DIR=$NOOP_HOME/build
hacer nanhu CORE_DIR=$NOOP_HOME/build

# generar el flujo de bits del FPGA
hacer que el flujo de bits CORE_DIR=$NOOP_HOME/build
```

Nota: El tiempo de generación es largo. Según el rendimiento de la máquina, se necesitan entre 8 y 12 horas para completar la generación de bits.

### Descargar prueba

Guión: [onboard-ai1-119.tcl](https://raw.githubusercontent.com/OpenXiangShan/env-scripts/main/fpga/onboard-ai1-119.tcl)

Imagen de memoria de instancia txt: [data.txt](https://raw.githubusercontent.com/OpenXiangShan/XiangShan-doc/main/docs/integration/resources/data.zip) (descargue el archivo zip y luego descomprímalo en txt) )

La primera ruta `<vivado_build_folder>` después de `-tclargs` es donde se almacenan los archivos bit y ltx generados por Vivado

``mierda
vivado -mode batch -source ./onboard-ai1-119.tcl -tclargs <carpeta_de_compilación_vivado> ./data.txt
```

Conectar al puerto serie, 115200, N, 8, 1

<font color=red>*Nombre de usuario: root*</font>

<font color=red>*Contraseña: bosc*</font>

Una vez cargado el sistema, podrá ver la siguiente información. Puede obtener la información actual de la CPU mediante `cat /proc/cpuinfo`:

![cpuinfo.png](../../figs/fpga_images/cpuinfo.png)

## Principio del sistema mínimo FPGA

### Interfaz principal de XiangShan

Las interfaces externas del núcleo de la CPU Xiangshan incluyen principalmente MEM AXI, DMA AXI y CFG AXI. Se utilizan para conectar al controlador DDR, la ruta de datos DMA y las operaciones IO respectivamente.

![fpga_minimal.png](../../figs/fpga_images/fpga_minimal.png)

![nanhu_interface.png](../../figs/fpga_images/nanhu_interface.png)

Cuando construimos el sistema mínimo, utilizamos principalmente las interfaces MEM AXI y CFG AXI porque no había una operación con gran rendimiento de datos.

### Tabla de mapeo de direcciones de XiangShan (CFG AXI)

A continuación se muestra la tabla de mapeo de direcciones de XiangShan. Al construir un sistema mínimo, solo necesitamos algunos de estos periféricos: `SYS_CFG`, `QSPI_FLASH` y `UART`. Por lo tanto, construimos un puente para conectarnos a CFG AXI en Vivado y sacamos los tres periféricos mencionados anteriormente de acuerdo con la tabla de mapeo de direcciones de XiangShan.

`QSPI_FLASH` almacena principalmente la ROM de arranque. En el proyecto Vivado, utilizamos bram con valores iniciales para simular.

`SYS_CFG` solo utiliza la información de la versión en el desplazamiento 0x0000_0000.

``cpp
#ifndef _XS_MEMMAP_H_
#define _XS_MEMMAP_H_

#define QSPI_FLASH_BASE 0x10000000

#define DMA_BASE 0x30040000
#define SDMMC_BASE 0x30050000
#define USB_BASE 0x30060000

#define QSPI_BASE 0x31000000
#define GMAC_BASE 0x31010000
#define HDMI_BASE 0x31020000
#define HDMI_PHY_BASE 0x31030000
#define DP_BASE 0x31040000

#define DDR0_BASE 0x31060000
#define DDR0_PHY_BASE 0x31070000

#define I2S_BASE 0x310A0000
#define UART0_BASE 0x310B0000
#define UART1_BASE 0x310C0000
#define UART2_BASE 0x310D0000
#define I2C0_BASE 0x310E0000
#define I2C1_BASE 0x310F0000
#define I2C2_BASE 0x31100000
#define I2C_BASE I2C0_BASE
#define GPIO_BASE 0x31110000
#define CRU_BASE 0x31120000
#define WDT_BASE 0x31130000

#define USB2_PHY0_BASE 0x31140000
#define USB2_PHY1_BASE 0x31150000
#define USB3_PHY0_BASE 0x31180000
#define USB3_PHY1_BASE 0x31190000
#define SOC_REG_BASE 0x31200000
#define PCIE0_CFG_BASE 0x311C0000
#define PCIE1_CFG_BASE 0x311D0000
#define SYSCFG_BASE 0x31200000
#define PMA_CFG0_BASE 0x38021000
#define PMA_CFG2_BASE 0x38021010
#define PMA_ADDR_BASE(x) (0x38021100 + 8 * x)

#define CPU_CFG_BASE 0x3a000000

#define PCIE0_BASE 0x40000000
#define PCIE1_BASE 0x60000000
#define DDR_BASE 0x80000000

#define UART_BASE UART0_BASE

#define la dirección de sincronización QSPI FLASH BASE + 0x07000000
#endif //_XS_MEMMAP_H_
```

![cfg_axi.png](../../figs/fpga_images/cfg_axi.png)

### Diseño de interfaz AXI de XiangShan MEM

La interfaz MEM AXI es relativamente simple y conecta principalmente XiangShan al controlador DDR. Se agregará un JTAG a AXI IP al FPGA para implementar la operación de carga del archivo de imagen de Linux a la dirección DDR.

![bloque_ddr.png](../../figs/fpga_images/bloque_ddr.png)

### Desde el script de descarga, vea el proceso de inicio

(1) Obtener la ruta del archivo de bits

``tcl
#parámetro
# primero: el pedacito de xiangshan
# segundo: datos de la carga de trabajo.txt
establecer xs_path [lindex $argv 0]
pone "ruta xiangshan:"
pone $xs_path
Establecer ruta_de_bit $xs_path/xs_fpga_top_debug.bit
Establecer ltx_path $xs_path/xs_fpga_top_debug.ltx
pone "bit_path:"
pone $bit_path
pone "ltx_path:"
pone $ltx_path
```

(2) Abra hw_server del servidor correspondiente y grabe el FPGA

``tcl
administrador_de_hw_abierto
conectar_servidor_hw -url localhost:3121 -permitir_no_jtag
objetivo_hw_actual [obtener_objetivos_hw */xilinx_tcf/Xilinx/00500208D0BAA]
establecer_propiedad PARAM.FRECUENCIA 10000000 [obtener_objetivos_hw */xilinx_tcf/Xilinx/00500208D0BAA]
objetivo_hw_abierto
establecer_propiedad SONDAS.ARCHIVO $ltx_path [obtener_dispositivos_hw xcvu19p_0]
establecer_propiedad SONDAS_COMPLETAS.ARCHIVO $ltx_path [obtener_dispositivos_hw xcvu19p_0]
conjunto_adecuado
