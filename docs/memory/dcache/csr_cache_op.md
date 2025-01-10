# Personalizar las operaciones de caché de primer nivel

La arquitectura de Southlake admite operaciones de caché L1 personalizadas basadas en CSR.

## Registro de instrucciones de caché L1

Los registros de instrucciones de caché se dividen en tres categorías: un registro de instrucciones de caché `CSR_CACHE_OP`, un registro de estado de instrucciones de caché `CSR_OP_FINISH` y `CSR_CACHE_*`: registros de control de caché de primer nivel. La configuración de parámetros de instrucciones de caché personalizadas y la devolución de resultados son todos Se realiza a través de estos registros. La dirección base del registro se especifica mediante `Sfetchctl`, y el valor predeterminado es `Sfetchctl`. La lista detallada de registros es la siguiente en orden:

Nombre del registro | Descripción
-|-
CSR_CACHE_OP|código de instrucción de operación de caché. Escribir en este registro activa la ejecución de instrucciones de caché L1 personalizadas.
CSR_OP_FINISH|Bandera de finalización de instrucción de caché L1
CACHE_LEVEL|Selección de destino de instrucción de caché. 0: ICache, 1: DCache
CACHE_WAY|selección de ruta de caché
CACHE_IDX|selección de índice de caché
CACHE_BANK_NUM|selección de banco de caché
CACHE_TAG_ECC|etiqueta ecc
reservado|reservado
CACHE_TAG_LOW|datos de etiqueta
CACHE_TAG_HIGH|reservado
reservado|reservado
CACHE_DATA_ECC|datos ecc
DATOS_CACHE_X|datos [64(X+1)-1:64(X)]

<!-- TODO: Reducir el espacio de codificación -->

## código de instrucción de caché

Los códigos de operación admitidos por las instrucciones de caché L1 personalizadas son los siguientes:

Operación|Código de operación
-|-
LEER_ETIQUETA_ECC|0
LEER_DATOS_ECC|1
LEER_ETIQUETA|2
LEER_DATOS|3
ESCRIBIR_ETIQUETA_ECC|4
ESCRIBIR_DATOS_ECC|5
ESCRIBIR_ETIQUETA|6
ESCRIBIR DATOS | 7
<!-- BLOQUE DE DESCARGA DE COP|8 -->
## Personalizar el flujo de ejecución básico de las instrucciones de caché L1

1. Utilice la instrucción CSR para escribir el registro de configuración de parámetros en el registro de instrucciones de caché y borrar el registro OP_FINISH.
1. Escriba el código de instrucción en el registro CSR_CACHE_OP
1. (Opcional) Sondee CSR_OP_FINISH hasta que se complete el comando, CSR_OP_FINISH == 1
1. Utilice la instrucción CSR para leer el registro de resultados en el registro de instrucciones de caché para obtener el resultado de la instrucción de caché.
