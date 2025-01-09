# Almacenamiento de datos

El módulo `DataStorage` es responsable del almacenamiento, lectura y escritura de datos de caché.
Según las características de cada canal Tilelink, el módulo `DataStorage` dispone de dos puertos de lectura (`sourceC_r`, `sourceD_r`).
3 puertos de escritura (`sinkD_w`, `sourceD_w`, `sinkC_w`),
Para mejorar la concurrencia de lectura y escritura, el módulo puede parametrizar el número de bancos internos, y diferentes bancos pueden leer y escribir en paralelo.
Además, el módulo también admite el overclocking parametrizado de los datos leídos desde la SRAM para lograr requisitos de frecuencia más altos.
