## 1. Instale [J-Link](https://baike.baidu.com/item/jlink%E4%BB%BF%E7%9C%9F%E5%99%A8/10410763#:~:text= Enlace J%E4%B8%BA%E5%BE%B7,%E4%BB%A5%E4%BF%9D%E9%9A%9C%E6%82%A8%E7%9A%84%E6%9D %83%E7%9B%8A%E3%80%82) (emulador JTAG desarrollado por SEGGER)

### 1.1 Instalar el paquete de software J-Link

1) Descargue el paquete deb [https://www.segger.com/downloads/jlink/](https://www.segger.com/downloads/jlink/)

2) sudo dpkg -i JLink_Linux_V750a_x86_64.deb

3) Verifique si está instalado: busque la ruta /opt/SEGGER/JLink/; ejecute JLinkExe en la terminal, si puede ingresar, es exitoso.

![estado usb](../../figs/fpga_images/usb_status.png)

![SEGGER JLink](../../figs/fpga_images/SEGGER_JLink.png)

![Versión de JLink](../../figs/fpga_images/JLink_version.png)

### 1.2 Sacar señales JTAG

El proyecto [sistema mínimo](https://github.com/OpenXiangShan/env-scripts) ya ha introducido las señales relacionadas con JTAG de la CPU. Si desea utilizar los periféricos de su propia plataforma FPGA, también debe modificar los correspondientes [archivo de restricción](https://github.com/OpenXiangShan/env-scripts/blob/main/xs_nanhu_fpga/src/constr/xiangshan.xdc), utilizado para que coincida con su propio entorno FPGA.

Si las restricciones relacionadas con JTAG son diferentes a las de nuestro entorno, deberá modificarlas según su propio entorno.

```
establecer_propiedad_PAQUETE_PIN F54 [obtener_puertos JTAG_TCK]
establecer_propiedad_PAQUETE_PIN A54 [obtener_puertos JTAG_TMS]
establecer_propiedad_PAQUETE_PIN H54 [obtener_puertos_JTAG_TDI]
establecer_propiedad PIN_DE_PAQUETE C54 [obtener_puertos JTAG_TDO]
establecer_propiedad_PAQUETE_PIN G54 [obtener_puertos JTAG_TRSTn]

establecer_propiedad IOSTANDARD LVCMOS18 [obtener_puertos JTAG_TCK]
establecer_propiedad IOSTANDARD LVCMOS18 [obtener_puertos JTAG_TMS]
establecer_propiedad IOSTANDARD LVCMOS18 [obtener_puertos JTAG_TDI]
establecer_propiedad IOSTANDARD LVCMOS18 [obtener_puertos JTAG_TDO]
establecer_propiedad IOSTANDARD LVCMOS18 [obtener_puertos JTAG_TRSTn]
```

## 2. Instalar el software OpenOCD

### 2.1 Descargar el repositorio de código fuente de OpenOCD

Descargar el código fuente de OpenOCD

[https://github.com/openocd-org/openocd.git](https://github.com/openocd-org/openocd.git)

``concha
clon de git https://github.com/openocd-org/openocd.git
```

### 2.2 Creación del entorno de compilación del código fuente de OpenOCD

``concha
./configure --prefix=/home/[**su ruta**]/openocd/riscv-openocd/openocd-bin --enable-verbose --enable-verbose-usb-io --enable-verbose-usb- comunicaciones --habilitar bitbang remoto --habilitar ftdi --deshabilitar werror --habilitar jlink --habilitar estático
```

### 2.3 Compilar el código fuente de OpenOCD

``concha
hacer
sudo make install
```

El resultado de la compilación se encuentra en ./riscv-openocd/openocd-bin/bin/openocd

### 2.4 Otros

Si encuentra el error de configuración: libjaylink-0.2 es necesario para el programador SEGGER J-Link durante la configuración

Significa que falta la dependencia libjaylink. Puedes instalar la biblioteca de dependencia siguiendo los pasos que se indican a continuación.

1. Introduzca la biblioteca libjaylink en el código openocd

``concha
cd src/jtag/drivers/libjaylink
```

2. libjaylink requiere autotools y libtool para compilar, así que asegúrese de tener estas herramientas instaladas:

``concha
sudo apt install automake autoconf libtool pkg-config
```

3. Configuración, compilación e instalación

``concha
./autogen.sh
./configurar
hacer
sudo make install
```

>Durante el proceso de configuración, asegúrese de que el USB no esté encendido, de lo contrario no se podrá encontrar jlink. Si el USB no está encendido debido a la falta de la biblioteca libusb, debe instalar la biblioteca correspondiente

![USB JTAG](../../figs/fpga_images/jtag_usb.png)

4. Instalar la biblioteca libusb

Vaya a [https://github.com/libusb/libusb/releases](https://github.com/libusb/libusb/releases)

Descargue el paquete de lanzamiento, como [libusb-1.0.27.tar.bz2](https://github.com/libusb/libusb/releases/download/v1.0.27/libusb-1.0.27.tar.bz2)

 Configuración, compilación e instalación

``concha
./configure --build=x86_64-linux --disable-udev
hacer
hacer instalación
```

## 3. Guía de uso

[https://openocd.org/doc/html/index.html](https://openocd.org/doc/html/index.html)

[https://openocd.org/doc/doxygen/html/index.html](https://openocd.org/doc/doxygen/html/index.html)

Problema 1: Habilitar --enable-ftdi Se produce el siguiente problema

configure: error: se requiere libusb-1.x para el modo MPSSE de dispositivos basados ​​en FTDI

Solución:

```
 sudo apt install libusb-1.0-0-dev
```
### 3.1 Grabe la imagen de hardware que admita la función JTAG

El hardware de producción está conectado a los bits correctos y se garantiza que el cableado externo sea correcto.

### 3.2 Iniciar OpenOCD (recuerde pasar el script de configuración de tiempo de ejecución como parámetro)

Versión de doble núcleo de Nanhu: [xs-jlink.config.dualcore.txt](https://raw.githubusercontent.com/OpenXiangShan/XiangShan-doc/main/docs/integration/resources/xs-jlink.config.dualcore. TXT)

```
./openocd-bin/bin/openocd -f ./xs-jlink.config.dualcore.txt
```

![salida_openocd](../../figs/fpga_images/salida_openocd.png)

### 3.3 Crear y compilar un subprograma de prueba

**Crear archivo de código C: hello.c**

```
carácter buf[0x100];

int principal(vacío)
{
 int f = 0x33;
 asm volátil("nop");
 asm volátil("li s0, 0x0");
 asm volátil("li sp, 0x80000100");
 asm volátil("addi sp, sp, -32");
 asm volátil("sd s0, 24(sp)");
 asm volátil("addi s0, sp, 32");
 asm volátil("nop");
 asm volátil("nop");
 asm volátil("j .");
 asm volátil("nop");
 f++;
 asm volátil("nop");
 mientras(f){}
 asm volátil("nop");
 asm volátil("nop");
 asm volátil("nop");
 asm volátil("nop");
 mientras(1){}
 devuelve 0;
}

```

**Crear script de enlace: hello.lds**

```
ARCHIVO_DE_SALIDA( "riscv" )

SECCIONES
{
 . = 0x80000000;
 .texto : { *(.texto) }
 . = 0x80000400;
 .datos : { *(.datos) }
}

```

**Compilación**

```
riscv64-unknown-linux-gnu-gcc -g -O0 -o hello.o -c hello.c # Generar archivo de destino
riscv64-unknown-linux-gnu-gcc -g -Og -T hello.lds -nostartfiles -o hello.elf hello.o # Generar archivo ejecutable elf
```

En este punto, se ha generado el archivo del programa de prueba hello.elf para cargar. A continuación, puede utilizar el comando GDB (load hello.elf) para cargarlo en la ubicación especificada en la memoria DDR para realizar pruebas. Además, copie también el archivo hello.c a la carpeta de archivos de trabajo de GDB para que la depuración de un solo paso posterior pueda mostrar información del código C.

Para escribir 8 bytes de datos, es necesario modificar los parámetros de compilación:

Línea 13 del siguiente código

```
#define REG32(addr) (*(volátil unsigned long *)(unsigned long)(addr))
#define read_csr(reg) ({ largo sin signo __tmp; \
 asm volátil ("csrr %0, " #reg : "=r"(__tmp)); \
 __tmp; })

#define write_csr(reg, val) ({ \
 asm volátil ("csrw " #reg ", %0" :: "rK"(val)); })

int principal(vacío)
{
 unsigned long pmp = 0;
 int datos = 0, i = 0;
 escribir
