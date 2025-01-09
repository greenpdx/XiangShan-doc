# Diseño del prefetcher L2

La caché L2 de XiangShan utiliza la precarga de hardware Best-Offset. BOP es un prefetch basado en offset. El offset se puede ajustar dinámicamente a medida que se ejecuta el programa. Su objetivo principal es garantizar la puntualidad del prefetch.

El algoritmo de precarga específico se puede encontrar en el artículo. A continuación, se presenta únicamente la parte relacionada con la implementación específica de XiangShan.

<!--
Es posible que necesites una imagen aquí
-->
En primer lugar, se utiliza un bit de prebúsqueda en el directorio L2 para registrar si un bloque de caché es un bloque prebúsquedo. Cuando MSHR recibe una solicitud de adquisición de DCache, si la dirección solicitada no llega a la capa 2 o llega al bloque precargado, MSHR iniciará una solicitud de "activación de precarga", que se enviará al precargador. El precargador generará una dirección de precarga basada en en la dirección de la solicitud y el mejor desplazamiento entrenado por el algoritmo Best-offset. Luego, el prefetcher enviará la solicitud de prefetch al banco donde se encuentra la dirección. El tipo de solicitud es Intent, el asignador MSHR asigna un MSHR para la solicitud de prefetch. Si el bloque de precarga no está en L2, MSHR será responsable de traer el bloque de precarga desde L3.

Cuando MSHR completa una búsqueda previa, envía otra respuesta al buscador previo, y el buscador previo entrena el algoritmo Best-offset basándose en esta respuesta.

<!--
TODO: ¡Recuerda citar! ! !
-->
