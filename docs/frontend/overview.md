# Documentación de la arquitectura front-end
Este capítulo describe la arquitectura general del frontend del procesador Xiangshan. La relación del módulo front-end y la ruta de datos se muestran en la siguiente figura:

![frontend](../figs/frontend/frontend.png)



La arquitectura Nanhu adopta una arquitectura de búsqueda de instrucciones que desacopla la predicción de bifurcaciones y la memoria caché de instrucciones. La unidad de predicción de bifurcaciones proporciona solicitudes de búsqueda de instrucciones y las escribe en una cola, que las envía a la unidad de búsqueda de instrucciones y a la memoria caché de instrucciones.
El código de instrucción obtenido se decodifica previamente para verificar inicialmente si hay errores de predicción de bifurcación y vaciar el flujo de predicción de manera oportuna. Las instrucciones verificadas se envían al búfer de instrucciones y se pasan al módulo de decodificación, lo que finalmente forma el suministro de instrucciones de back-end.

Este capítulo incluye las siguientes partes:

* [Predicción de rama](bp.md)
* [Instrucción de búsqueda de destino de la cola](ftq.md)
* [Unidad de obtención de instrucciones](ifu.md)
* [Caché de instrucciones](icache.md)
* [Unidad de decodificación](decode.md)

!!! nota "Acerca de las abreviaturas"
 Algunos sustantivos aparecerán en el documento en forma abreviada (aparecerán subrayados debajo). Puedes ver el nombre completo de la abreviatura al pasar el ratón sobre ellos.
