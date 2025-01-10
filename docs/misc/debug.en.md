Debug Xiangshan with GDB
====================

We connected Rocket Chip's Debug Module to Xiangshan to support the debug mode in version 0.13 [RISCV Debug Manual](https://riscv.org/wp-content/uploads/2019/03/riscv-debug-release.pdf).

## Principle

The Debug Module can make the processor core enter debug mode through interrupts, and then GDB can make the processor core execute any instructions and read any registers. After the Debug Module issues an interrupt, the processor core will fall into the MMIO address space of the Debug Module and start a loop. The Debug Module controls the behavior of the core and reads the registers of the core by writing instructions in this address space.

The Debug Module also supports the System Bus Access function, that is, reading and writing physical memory without going through the processor core. Xiangshan's Debug Module is connected to the L3 cache through TileLink to facilitate data consistency maintenance. Please note that to use this function during simulation, please use the Default Config.

## Software and Configuration
To debug Xiangshan through GDB, you need to install `riscv64-unknown-elf-gdb` and `openocd`. For installation methods, please refer to the corresponding Github projects (directly follow the README to install, no additional configuration is required): [GDB](https://github.com/riscv-collab/riscv-binutils-gdb), [OpenOCD](https://github.com/riscv/riscv-openocd)

OpenOCD requires a configuration file, and the following configuration can be used:

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

## Connect to Xiangshan with GDB
After compiling emu, add the two flags `--enable-jtag --no-diff` when running, and execute `openocd -f config.cfg` in another terminal, where `config.cfg` is the OpenOCD configuration file, and execute `riscv64-unknown-elf-gdb` in a third terminal. It is recommended to use tmux split screen for operation.

Execute `target extended-remote localhost:3333` in the GDB command line, and wait a while to connect to the Xiangshan emulator.
