# Guía DFT

Esta sección describe los requisitos básicos de DFT.

## Diseño de bus compartido 

El flujo de diseño tradicional de inserción de MBIST alrededor de la memoria ha resaltado cada vez más la contradicción de PPA (rendimiento, potencia, área) en procesadores de alto rendimiento. Para evitar afectar la sincronización de funciones tanto como sea posible, reduzca el costo de área de MBIST y Para reducir el consumo de energía de prueba de MBIST, NANHU adopta la interfaz de bus MBIST basada en sharedbus.<br>En los procesadores de alto rendimiento, el proceso de diseño tradicional de insertar MBIST alrededor de la memoria resalta cada vez más el PPA (rendimiento, consumo de energía, área). ) ) contradicción. Para minimizar el impacto en la sincronización funcional, reducir el costo de área de MBIST y reducir el consumo de energía de prueba de MBIST, Nanhu adopta una interfaz de bus MBIST basada en un bus compartido.
