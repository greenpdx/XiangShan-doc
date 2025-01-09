# Guía de síntesis lógica

En esta sección se describe la información necesaria para sintetizar el diseño de NANHU.<br>Esta sección describe la información necesaria para sintetizar el diseño de NANHU.

## Reloj y reinicio Reloj y reinicio

Consulte la sección SoC para conocer las configuraciones de reloj y reinicio.

## Sincronización de SRAM de caché L3
La caché L3 en NANHU está configurada de forma predeterminada para ser accedida (leer o escribir) una vez cada dos ciclos.

* Escritura SRAM Escritura SRAM

![](../figs/integration/sram_write.png)

* Lectura de SRAM Lectura de SRAM

Existen dos casos posibles para la lectura de SRAM, dependiendo de la relación entre la latencia de lectura de la propia SRAM y la frecuencia del procesador. Aunque el comportamiento de la SRAM en dos situaciones es diferente, ambas son correctas ya que la caché L3 solo utiliza los datos rdata después de la lectura. dos ciclos.<br>Dependiendo de la relación entre la latencia de lectura de la propia SRAM y la frecuencia del procesador, puede haber dos situaciones para la lectura de la SRAM. Aunque la SRAM se comporta de manera diferente en los dos casos, ambos son correctos porque la caché L3 utiliza los datos leídos solo dos ciclos después.

Caso 1: Latencia (SRAM) < Tiempo de ciclo (procesador) Latencia de SRAM < tiempo de ciclo del procesador

![](../figs/integration/sram_read_1.png)

Caso 2: Latencia (SRAM) > Tiempo de ciclo (procesador) Latencia de SRAM > ciclo del procesador

![](../figs/integration/sram_read_2.png)

* Lectura y escritura de SRAM<br>Lectura y escritura de SRAM

Caso 1: Latencia (SRAM) < Tiempo de ciclo (procesador) Latencia de SRAM < tiempo de ciclo del procesador

![](../figs/integracion/sram_rw_1.png)

Caso 2: Latencia (SRAM) > Tiempo de ciclo (procesador) Latencia de SRAM > ciclo del procesador

![](../figs/integracion/sram_rw_2.png)
