## 1. Install [J-Link](https://baike.baidu.com/item/jlink%E4%BB%BF%E7%9C%9F%E5%99%A8/10410763#:~:text=J-Link%E4%B8%BA%E5%BE%B7,%E4%BB%A5%E4%BF%9D%E9%9A%9C%E6%82%A8%E7%9A%84%E6%9D%83%E7%9B%8A%E3%80%82) (JTAG emulator developed by SEGGER)

### 1.1 Install J-Link software package

1) Download the deb package [https://www.segger.com/downloads/jlink/](https://www.segger.com/downloads/jlink/)

2) sudo dpkg -i JLink_Linux_V750a_x86_64.deb

3) Check whether it is installed: find the path /opt/SEGGER/JLink/; execute JLinkExe in the terminal, if it can enter, it is successful.

![usb status](../../figs/fpga_images/usb_status.png)

![SEGGER JLink](../../figs/fpga_images/SEGGER_JLink.png)

![JLink version](../../figs/fpga_images/JLink_version.png)

### 1.2 Lead out JTAG signals

The [minimum system](https://github.com/OpenXiangShan/env-scripts) project has already led out the CPU JTAG related signals. If you want to use the peripherals of your own FPGA platform, you also need to modify the corresponding [constraint file](https://github.com/OpenXiangShan/env-scripts/blob/main/xs_nanhu_fpga/src/constr/xiangshan.xdc) to match your own FPGA environment.

If the JTAG-related constraints are different from our environment, you need to modify them according to your own environment.

``` set_property PACKAGE_PIN F54 [get_ports JTAG_TCK] set_property PACKAGE_PIN A54 [get_ports JTAG_TMS] set_property PACKAGE_PIN H54 [get_ports JTAG_TDI] set_property PACKAGE_PIN C54 [get_ports JTAG_TDO] set_property PACKAGE_PIN G54 [get_ports JTAG_TRSTn] set_property IOSTANDARD LVCMOS18 [get_ports JTAG_TCK] set_property IOSTANDARD LVCMOS18 [get_ports JTAG_TMS] set_property IOSTANDARD LVCMOS18 [get_ports JTAG_TDI] set_property IOSTANDARD LVCMOS18 [get_ports JTAG_TDO]
set_property IOSTANDARD LVCMOS18 [get_ports JTAG_TRSTn]
```

## 2. Install OpenOCD software

### 2.1 Download OpenOCD source code repository

Download openocd source code

[https://github.com/openocd-org/openocd.git](https://github.com/openocd-org/openocd.git)

```shell
git clone https://github.com/openocd-org/openocd.git
```

### 2.2 Build OpenOCD source code compilation environment

```shell
./configure --prefix=/home/[**your path**]/openocd/riscv-openocd/openocd-bin --enable-verbose --enable-verbose-usb-io --enable-verbose-usb-comms --enable-remote-bitbang --enable-ftdi --disable-werror --enable-jlink --enable-static
```

### 2.3 Compile OpenOCD source code

```shell
make
sudo make install
```

The compilation result is located in ./riscv-openocd/openocd-bin/bin/openocd

### 2.4 Others

If you encounter configure: error: libjaylink-0.2 is required for the SEGGER J-Link Programmer during configuration, it means that libjaylink dependencies are missing. You can follow the steps below to install the dependent library

1. Enter the libjaylink library in the openocd code

```shell
cd src/jtag/drivers/libjaylink
```

2. libjaylink requires autotools and libtool to compile, so make sure you have these tools installed:

```shell
sudo apt install automake autoconf libtool pkg-config
```

3. Configure, compile and install

```shell
./autogen.sh
./configure
make
sudo make install
```

>During the configuration process, make sure that USB is not turned on, otherwise jlink cannot be found. If the USB is not turned on due to the lack of libusb library, you need to install the corresponding library

![JTAG USB](../../figs/fpga_images/jtag_usb.png)

4. Install libusb library

Enter [https://github.com/libusb/libusb/releases](https://github.com/libusb/libusb/releases)

Download the release package, such as [libusb-1.0.27.tar.bz2](https://github.com/libusb/libusb/releases/download/v1.0.27/libusb-1.0.27.tar.bz2)

Configure, compile and install

```shell
./configure--build=x86_64-linux --disable-udev
make
make install
```

## 3. Usage Guide

[https://openocd.org/doc/html/index.html](https://openocd.org/doc/html/index.html)

[https://openocd.org/doc/doxygen/html/index.html](https://openocd.org/doc/doxygen/html/index.html)

Problem 1: Enable --enable-ftdi The following problem

configure: error: libusb-1.x is required for the MPSSE mode of FTDI based devices

Solution:

```
sudo apt install libusb-1.0-0-dev
```
### 3.1 Burn the hardware image that supports JTAG function

Connect the correct bit of the production hardware and ensure that the external connection is correct

### 3.2 Start OpenOCD (remember to pass the runtime configuration script as a parameter)

Nanhu Dual-core version: [xs-jlink.config.dualcore.txt](https://raw.githubusercontent.com/OpenXiangShan/XiangShan-doc/main/docs/integration/resources/xs-jlink.config.dualcore.txt)

```
./openocd-bin/bin/openocd -f ./xs-jlink.config.dualcore.txt
```

![openocd_output](../../figs/fpga_images/openocd_output.png)

### 3.3 Create and compile a small test program

**Create a c code file: hello.c**

```
char buf[0x100];

int main(void)
{
int f = 0x33;
asm volatile("nop");
asm volatile("li s0, 0x0");
asm volatile("li sp, 0x80000100"); asm volatile("addi sp, sp, -32"); asm volatile("sd s0, 24(sp)"); asm volatile("addi s0, sp, 32"); asm volatile("nop"); asm volatile("nop"); asm volatile("j ."); asm volatile("nop"); f++; asm volatile("nop"); while(f){} asm volatile("nop"); asm volatile("nop"); asm volatile("nop"); asm volatile("nop"); while(1){} return 0; } ``` **Create link script: hello.lds** ``` OUTPUT_ARCH( "riscv" )

SECTIONS
{
. = 0x80000000;
.text : { *(.text) }
. = 0x80000400;
.data : { *(.data) }
}

```

**Compile**

```
riscv64-unknown-linux-gnu-gcc -g -O0 -o hello.o -c hello.c # Generate target file
riscv64-unknown-linux-gnu-gcc -g -Og -T hello.lds -nostartfiles -o hello.elf hello.o # Generate elf executable file
```

The test program file hello.elf for loading has been generated. You can use the GDB command (load hello.elf) to load it to the specified location in the DDR memory for testing. In addition, please copy the hello.c file to the GDB working folder. So that the subsequent single-step debugging can display C code information.

It involves writing 8 bytes of data, so the compilation parameters need to be modified:

The following code has 13 lines

```
#define REG32(addr) (*(volatile unsigned long *)(unsigned long)(addr))
#define read_csr(reg) ({ unsigned long __tmp; \
asm volatile ("csrr %0, " #reg : "=r"(__tmp)); \
__tmp; })

#define write_csr(reg, val) ({ \
asm volatile ("csrw " #reg ", %0" :: "rK"(val)); })

int main(void)
{
unsigned long pmp = 0;
int data = 0, i = 0;
write
