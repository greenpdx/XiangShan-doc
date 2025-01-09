# Unidad de decodificación
## Funcionalidad básica
Las instrucciones se extraen de la memoria caché de instrucciones, se envían al búfer de instrucciones (cola) para su almacenamiento temporal y luego se envían a la unidad de decodificación para su decodificación a una velocidad de 6 por ciclo, y luego pasan a la siguiente etapa de la cadena de procesamiento.

## Instrucción Fusión
La unidad de decodificación admite la fusión de algunas instrucciones. En el módulo `FusionDecoder`, la fusión de instrucciones se realiza en función de los datos de 32 bits de dos instrucciones consecutivas. Actualmente, Xiangshan no admite la fusión de instrucciones para dos instrucciones no contiguas.

Actualmente, Xiangshan admite la siguiente fusión de comandos:

* limpia los 32 bits superiores / obtiene los 32 bits inferiores: `slli r1, r0, 32` + `srli r1, r1, 32`
* limpia los 48 bits superiores / obtiene los 16 bits inferiores: `slli r1, r0, 48` + `srli r1, r1, 48`
* limpia los 48 bits superiores / obtiene los 16 bits inferiores: `slliw r1, r0, 16` + `srliw r1, r1, 16`
* signo-extensión de un número de 16 bits: `slliw r1, r0, 16` + `sraiw r1, r1, 16`
* desplazar a la izquierda uno a uno y agregar: `slli r1, r0, 1` + `add r1, r1, r2`
* desplazar a la izquierda dos veces y agregar: `slli r1, r0, 2` + `add r1, r1, r2`
* desplazar a la izquierda tres veces y agregar: `slli r1, r0, 3` + `add r1, r1, r2`
* desplaza la palabra extendida a cero a la izquierda en uno: `slli r1, r0, 32` + `srli r1, r0, 31`
* desplaza la palabra extendida a cero hacia la izquierda por dos: `slli r1, r0, 32` + `srli r1, r0, 30`
* desplaza la palabra extendida a cero a la izquierda por tres: `slli r1, r0, 32` + `srli r1, r0, 29`
* obtener el segundo byte: `srli r1, r0, 8` + `andi r1, r1, 255`
* desplazar a la izquierda cuatro veces y agregar: `slli r1, r0, 4` + `add r1, r1, r2`
* desplazar a la derecha 29 y agregar: `srli r1, r0, 29` + `add r1, r1, r2`
* desplazar a la derecha 30 y agregar: `srli r1, r0, 30` + `add r1, r1, r2`
* desplazar a la derecha 31 y agregar: `srli r1, r0, 31` + `add r1, r1, r2`
* desplazar a la derecha 32 y agregar: `srli r1, r0, 32` + `add r1, r1, r2`
* agrega uno si es impar, de lo contrario no cambia: `andi r1, r0, 1` + `add r1, r1, r2`
* agregue uno si es impar (en formato Word), de lo contrario no cambia: `andi r1, r0, 1` + `addw r1, r1, r2`
* `addw` y extrae sus 8 bits inferiores (fusionados en `addwbyte`)
* `addw` y extrae su bit inferior (fusionado en `addwbit`)
* `addw` y `zext.h` (fusionados en `addwzexth`)
* `addw` y `sext.h` (fusionados en `addwsext`)
* operación lógica y extraer su LSB
* operación lógica y extraer sus 16 bits inferiores
* `O(Gato(origen1(63, 8), 0.U(8.W)), origen2)`
* mul de datos de 7 bits con datos de 32 bits
