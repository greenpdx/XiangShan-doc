# Introducción y guía del proyecto Xiangshan

## 1. ¿Qué es Xiangshan?
Xiangshan es un procesador RISC-V de código abierto y alto rendimiento basado en el lenguaje de diseño de hardware Chisel y admite el conjunto de instrucciones RV64GC. Durante el desarrollo del procesador Xiangshan, se utilizó una gran cantidad de herramientas de código abierto, incluidas Chisel, Verilator y GTKwave, para implementar herramientas básicas para el desarrollo del procesador, como verificación diferencial, instantáneas de simulación y puntos de control RISC-V, estableciendo un conjunto de Procesos de desarrollo de procesadores ágiles basados ​​en herramientas de código abierto, incluido el diseño, la implementación y la verificación.

**Enlace del proyecto Xiangshan**：

- GitHub：[https://github.com/OpenXiangShan](https://github.com/OpenXiangShan)

- 码云：[https://gitee.com/OpenXiangShan](https://gitee.com/OpenXiangShan)


**Introducción al almacén del proyecto Xiangshan**：

[**XiangShan**](https://github.com/OpenXiangShan/XiangShan)：Implementación del procesador Xiangshan

[**XiangShan-doc**](https://github.com/OpenXiangShan/XiangShan-doc)：Documentos del procesador Xiangshan, incluidos documentos de diseño, informes públicos y aclaraciones de noticias erróneas.
[**NEMU**](https://github.com/OpenXiangShan/NEMU/tree/master)：Un intérprete ISA de alto rendimiento con una eficiencia cercana a QEMU, aquí hay uno[Vídeo de introducción](https://www.bilibili.com/video/BV1Zb4y1k7RJ)

[**nexus-am**](https://github.com/OpenXiangShan/nexus-am)：Máquina abstracta, que proporciona el entorno de ejecución para los programas. Hay una[Breve introducción](https://nju-projectn.github.io/ics-pa-gitbook/ics2020/2.3.html)

[**DRAMsim3**](https://github.com/OpenXiangShan/DRAMsim3)：Simular el comportamiento de la memoria a nivel de ciclo

[**env-scripts**](https://github.com/OpenXiangShan/env-scripts)：Algunos archivos de script, incluido análisis de rendimiento, reemplazo de módulo y análisis de tiempo, etc.

Y otros repositorios incluyen **riscv-linux**, **riscv-pk**, **riscv-torture** y así sucesivamente.

## 2. Estructura del directorio del procesador Xiangshan

```
.
├── debug                # Algunos comandos comunes para ejecutar pruebas se escriben en forma de scripts.
├── scripts              # Generar Verilog y algunos scripts para simulación
├── src                  # Código de diseño y verificación estructural
│   ├── main               # Código de diseño estructural
│   │   └── scala
│   │       ├── bus            # Algunas herramientas de bus
│   │       ├── device         # AL comparte Nash y RAM IE y dijo que no
│   │       ├── difftest       # Descripción de la interfaz de prueba diferencial
│   │       ├── system         # Descripción de SoC
│   │       ├── top            # Archivo de nivel superior
│   │       ├── utils          # Algunas bibliotecas de herramientas de hardware básicas
│   │       ├── xiangshan      # El código de diseño de la pieza de CPU Xiangshan
│   │       └── xstransforms   # Algunas transformaciones de FIRRTL
│   └── test               # Código de verificación
│       ├── csrc             # Código C/C++ para simulación
│       │   ├── common         # Componentes comunes en la simulación
│       │   ├── difftest       # Implementación de la interfaz C/C++ de prueba diferencial
│       │   ├── vcs            # Archivos utilizados durante la simulación VCS
│       │   └── verilator      # Archivos utilizados por la simulación de Verilator
│       ├── scala            # Algunas pruebas unitarias basadas en ChiselTest
│       │   ├── cache          # Prueba de unidad de caché
│       │   ├── top
│       │   └── xiangshan      # Prueba de la unidad del módulo interno del núcleo de Xiangshan
│       ├── testcase
│       │   ├── Makefile
│       │   └── tests
│       └── vsrc             # Código Verilog para simulación
│           ├── common
│           └── vcs
├── ready-to-run         # Biblioteca de enlaces dinámicos nemu precompilada y algunas cargas
├── rocket-chip          # Se utiliza para obtener el marco de Diplomacia (pendiente de división ascendente)
├── berkeley-hardfloat   # Flotador duro modificado, actualmente utilizado como FPU de Xiangshan
├── block-inclusivecache # El SiFive InclusiveCache modificado se utiliza actualmente como L2 y L3 de Xiangshan
└── chiseltest           # Código fuente del marco ChiselTest
```

## 3. Diseño de la arquitectura del procesador Xiangshan

El procesador Xiangshan está diseñado con una estructura de seis elementos fuera de orden. El front-end incluye una unidad de búsqueda de instrucciones, una unidad de predicción de bifurcaciones, un búfer de instrucciones y otras unidades, y busca instrucciones de forma secuencial. El back-end incluye decodificación, cambio de nombre, reordenamiento de buffer, estación de reserva, archivo de registro de punto flotante/entero y unidad de operación de punto flotante/entero. Separamos el subsistema de acceso a la memoria, incluyendo dos pipelines de carga y dos pipelines de almacenamiento, así como colas de carga y colas de almacenamiento independientes, Store Buffer, etc. La caché incluye módulos como I\$, D\$, L1+\$, L2\$, TLB y prefetcher. La posición de cada parte en la tubería y la configuración de los parámetros se pueden obtener de la siguiente figura.

![Diagrama de la estructura del procesador Xiangshan](images/xs-arch-simple.svg)

Más adelante se presenta el diseño estructural específico del procesador Xiangshan.

## 4. Pruebas unitarias

Implementamos el marco de pruebas de caché Agent-Faker basado en ChiselTest. Su concepto de diseño es similar a UVM y actualmente se aplica a Dcache, L2Cache y TLB.

![Agent-Faker](images/AgentFaker.png)

## 5. Prueba de simulación

El procesador Xiangshan utiliza principalmente verilator para simulación y simula periféricos como uart y tarjeta SD. Durante la simulación, se realiza una comparación del tiempo de ejecución con el simulador NEMU, y se pasa información como la interrupción del reloj del procesador al simulador, lo que guía al simulador para que sea coherente con el procesador en las elecciones clave, mejorando así la flexibilidad de la simulación.

![Prueba de diferencias](images/difftest.png)

Cuando los comportamientos del procesador y del simulador son inconsistentes, el programa de simulación se detendrá y se podrá realizar un análisis de errores visualizando formas de onda y registros. Desarrollamos Wave Terminator para extraer formas de onda con semántica de bajo nivel en registros con semántica de alto nivel. Además, también hemos desarrollado un visor de registros [LogViewer](https://github.com/OpenXiangShan/env-scripts/tree/main/logviewer) para facilitar la visualización de los registros.

## 6. Evaluación del desempeño

Cada módulo puede imprimir contadores libremente durante la simulación, con la opción de imprimir contadores en tiempo real y al final de la simulación. Los contadores de rendimiento se analizan mediante la [herramienta de script](https://github.com/OpenXiangShan/env-scripts/blob/main/perf/perf.py). También desarrollamos una herramienta de visualización para analizar los contadores de rendimiento.

Para probar el rendimiento de los puntos de referencia SPEC y realizar análisis de rendimiento en programas SPEC, se utiliza NEMU para generar SimPoint de programas SPEC, realizar pruebas paralelas a gran escala y obtener rápidamente puntajes SPEC. Además, los fragmentos SPEC se pueden utilizar para evaluar el rendimiento durante el desarrollo.
