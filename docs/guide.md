#Guía del proyecto Xiangshan

## Enlace del proyecto Xiangshan

- GitHub: [https://github.com/OpenXiangShan](https://github.com/OpenXiangShan)

- Gitee: [https://gitee.com/OpenXiangShan](https://gitee.com/OpenXiangShan)

- Enlace de Git: [https://gitlink.org.cn/OpenXiangShan/XiangShan](https://gitlink.org.cn/OpenXiangShan/XiangShan)

## Enlace al documento de Xiangshan

- Introducción al núcleo del procesador: [https://xiangshan-doc.readthedocs.io/zh_CN/latest/arch/](https://xiangshan-doc.readthedocs.io/zh_CN/latest/arch/)

- Marco de desarrollo ágil

 - Instrucciones de uso: [https://xiangshan-doc.readthedocs.io/zh_CN/latest/tools/xsenv/](https://xiangshan-doc.readthedocs.io/zh_CN/latest/tools/xsenv/)

 - Introducción a la herramienta: [https://xiangshan-doc.readthedocs.io/zh_CN/latest/tools/difftest/](https://xiangshan-doc.readthedocs.io/zh_CN/latest/tools/difftest/)

 - Solución de problemas: [https://xiangshan-doc.readthedocs.io/zh_CN/latest/tools/troubleshoot/](https://xiangshan-doc.readthedocs.io/zh_CN/latest/tools/troubleshoot/)

- Compilación de la cadena de herramientas: [https://xiangshan-doc.readthedocs.io/zh_CN/latest/compiler/gnu_toolchain/](https://xiangshan-doc.readthedocs.io/zh_CN/latest/compiler/gnu_toolchain/)

## Introducción al almacén del proyecto Xiangshan

El código RTL de Xiangshan en sí:

* [**XiangShan**](https://github.com/OpenXiangShan/XiangShan): Implementación del procesador XiangShan

* [**HuanCun**](https://github.com/OpenXiangShan/HuanCun): caché L2/L3 sin bloqueo para procesadores XiangShan

* [**FuDian**](https://github.com/OpenXiangShan/fudian): Unidad de punto flotante del procesador Xiangshan

Entorno de simulación de Xiangshan:

* [**Difftest**](https://github.com/OpenXiangShan/XiangShan): Marco de co-simulación de procesador utilizado por XiangShan

* [**NEMU**](https://github.com/OpenXiangShan/NEMU/tree/master): Un intérprete ISA de alto rendimiento con una eficiencia cercana a la de QEMU. Aquí hay un [video de introducción](https:// (www.bilibili.com/video/BV1Zb4y1k7RJ)

* [**nexus-am**](https://github.com/OpenXiangShan/nexus-am): máquina abstracta que proporciona el entorno de ejecución del programa. Aquí hay una [breve introducción](https://nju -proyecton.github.io/ics-pa-gitbook/ics2020/2.3.html)

* [**DRAMsim3**](https://github.com/OpenXiangShan/DRAMsim3): Simulación a nivel de ciclo del comportamiento de la memoria, configurada para el proyecto Xiangshan

* [**env-scripts**](https://github.com/OpenXiangShan/env-scripts): algunos archivos de script, incluido análisis de rendimiento, reemplazo de módulo y análisis de tiempo, etc.

* [**xs-env**](https://github.com/OpenXiangShan/xs-env): secuencia de comandos de implementación del entorno de desarrollo del frontend del procesador Xiangshan

Documentos de Xiangshan:

* [**XiangShan-doc**](https://github.com/OpenXiangShan/XiangShan-doc) : Documentación del procesador XiangShan, incluidos documentos de diseño, informes públicos y aclaraciones sobre noticias erróneas.

Otros repositorios bajo el proyecto Xiangshan incluyen **riscv-linux**, **riscv-pk**, **riscv-torture**, etc.

## Estructura del directorio del procesador Xiangshan

[**XiangShan**](https://github.com/OpenXiangShan/XiangShan) La estructura de directorios del almacén principal es la siguiente:

```
.
├── scripts   # Generar algunos scripts para Verilog y simulación
├── src       # Código de diseño estructural y verificación
│   └── principal # Código de diseño estructural
│      └── escala
│      ├── dispositivo # Algunos periféricos utilizados para simulación
│ ├── sistema # Descripción del SoC
│      ├── top # Archivo de nivel superior
│ ├── utils # Algunas bibliotecas de herramientas de hardware básicas
│      ├── xiangshan # Código de diseño de la parte de CPU de Xiangshan
│      └── xstransforms # Algunas transformaciones FIRRTL
├── fudian # Submódulo de punto flotante de Xiangshan
├── huancun # Submódulo de caché L2/L3 de Xiangshan
├── difftest # Marco de simulación colaborativa de Xiangshan
├── lista para ejecutar # Biblioteca de vínculos dinámicos nemu precompilada y algunas cargas útiles
└── rocket-chip # Se utiliza para obtener el marco de Diplomacia (esperando la división ascendente)
```
