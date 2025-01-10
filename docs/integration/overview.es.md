# Descripción general del núcleo de la CPU 

XiangShan-2 (NANHU) admite configuraciones de un solo núcleo y de dos núcleos, donde cada núcleo tiene su propia caché L1/L2 privada. L3 es compartida por varios núcleos. 

Tres interfaces AXI se comunican con la parte no central, incluido el puerto de memoria, el puerto DMA y el puerto periférico. También tiene interfaces de reloj, reset y JTAG. Consulte la Guía de integración para obtener más detalles.

NANHU apunta a 2 GHz a 14 nm y a 2,4 GHz ~ 2,8 GHz a 7 nm.
## Configuraciones típicas
A continuación se muestran las configuraciones típicas del núcleo NANHU:

| Artículo destacado | NANHU (XiangShan-2) |
| ------- | ------------------- |
| Número de etapa del ducto | 11 |
| Ancho del decodificador | 6 |
| Cambiar nombre ancho | 6 |
| Buffer de reordenamiento | 256 |
| Ancho del registro físico | 192(entero), 192(flotante) |
| Cola de carga | 80 |
| Cola de almacenamiento| 64|
| Caché de instrucciones L1 | 64 KB/128 KB (4/8 vías) |
| Caché de datos L1 | 64 KB/128 KB (4/8 vías) |
| Caché L2 | 512 KB/1 MB, 8 vías, no inclusivo |
| Caché L3 | 2 MB ~ 8 MB, 8 vías, no inclusivo |
| Tamaño del archivo de registro físico | 192x64 bits, 14R8W |
| Soporte ECC | Y |
| Soporte de memoria virtual | Y |
| Protección de la memoria física | Y |
| Virtualización | N |
| Vectorización | N |

## Soporte ISA Soporte ISA

| Conjunto de instrucciones | Descripción |
| ------- | ------------------- |
| I | Instrucciones para números enteros |
| M | Instrucciones de multiplicación y división de enteros |
| A | Instrucciones atómicas |
| F | Instrucción de punto flotante de precisión simple |
| D | Instrucciones de punto flotante de doble precisión |
| C | Instrucciones comprimidas de 16 bits |
| Zba | Extensión Bitmanip - instrucciones para generar direcciones |
| Zbb | Extensión Bitmanip - instrucciones básicas de manipulación de bits |
| Zbc | Extensión Bitmanip - instrucción de multiplicación sin acarreo |
| Zbs | Extensión Bitmanip - instrucciones de un solo bit |
| zbkb | Extensiones de criptografía - Instrucciones Bitmanip |
| Zbkc | Extensiones de criptografía - Instrucciones de multiplicación sin acarreo |
| zbkx | Extensiones de criptografía - Instrucciones de permutación de barras cruzadas |
| zknd | Extensiones de criptografía - Instrucciones de descifrado AES |
| zkne | Extensiones de criptografía - Instrucciones de cifrado AES |
| zknh | Extensiones AES - Instrucciones de función hash |
| zksed | Extensiones de criptografía - Instrucciones de cifrado de bloques SM4 |
| zksh | Extensiones de criptografía - Instrucciones de función hash SM3 |
| svinval | Instrucción de invalidación de caché de traducción de direcciones de grano fino |

## Latencia de instrucción

La mayoría de las instrucciones aritméticas son de ciclo único (`Latencia = 1`).
Las instrucciones de ciclo múltiple se enumeran a continuación.

| Instrucción(es) / Operaciones | Latencia | Descripciones |
|-------------- |------- | ------------ |
| `LD` | 4 (a ALU y LD), 5 (otros) | Operaciones de carga (para usar) |
| `MUL` | 3 | Multiplicación de enteros |
| `DIV` | 4~20 Divisor de enteros (SRT16) |
| `FMA` | 5 | Instrucción de multiplicación y suma de punto flotante (FMA en cascada)|
| `FADD`, `FMUL` | 3 | Operaciones de suma/multiplicación de punto flotante |
| `FDIV/SQRT` | 3~18 | Operaciones de división/raíz cuadrada de punto flotante |
| `CLZ(W)`, `CTZ(W)`, `CPOP(W)`, `XPERM(8/4)`, `CLMUL(H/R)` | 3 | Operaciones complejas de manipulación de bits|
| `AES64*`, `SHA256*`, `SHA512*`, `SM3*`, `SM4*` | 3 | Operaciones criptográficas escalares complejas |

## Modo de privilegio Nivel de privilegio

NANHU admite tres niveles de modo de privilegio: máquina (M), supervisor (S) y usuario (U).

## Microarquitectura

Consulte la sección Núcleo de CPU para obtener más detalles.

## Controlador de caché Controlador de caché

Hay un controlador de caché conectado a la caché L3, que se utiliza para realizar la operación de mantenimiento de caché (CMO). Los programadores deben utilizar el acceso a la memoria basado en MMIO para activar la operación requerida. Realiza una operación de mantenimiento de caché (CMO). El programador debe utilizar accesos a la memoria basados ​​en MMIO para activar las operaciones deseadas.

La siguiente es una tabla de registros del controlador de caché L3.

| Dirección | Ancho | Atr. | Descripción |
|------ | ----- | ----- | ----------- |
| 0x3900_0100 | 8B | RW | Registro `Tag` del bloque de caché de intereses |
| 0x3900_0108 | 8B | RW | Registro `Set` del bloque de caché de intereses |
| 0x3900_0110 | 8B | RW | Registro `Way` del bloque de caché de intereses (obsoleto) |
| 0x3900_0118 - 0x3900_0150 | 64B en total | RW | Registro `Data` del bloque de caché de intereses (obsoleto) |
