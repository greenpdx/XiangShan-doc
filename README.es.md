# XiangShan-doc
Documentación de XiangShan

Este es el repositorio de documentos oficial de Xiangshan.

* La documentación de la microestructura de Xiangshan se encuentra alojada en la plataforma ReadTheDocs. Le invitamos a visitar https://xiangshan-doc.readthedocs.io

* El directorio de diapositivas contiene nuestros informes técnicos, incluido el informe de la Cumbre de China RISC-V 2021, etc.



## Contáctenos

Lista de correo: xiangshan-all AT ict DOT ac DOT cn

Lista de correo: xiangshan-all AT ict DOT ac DOT cn



## Síganos

- **Cuenta oficial de WeChat: Procesador de código abierto Xiangshan**

![wechat](articulos/wechat.png)

- **Zhihu: [Procesador de código abierto Xiangshan](https://www.zhihu.com/people/openxiangshan)**

- **Weibo: [Procesador de código abierto Xiangshan](https://weibo.com/u/7706264932)**



## Se necesitan traducciones

Necesitamos traducir la mayor parte de la documentación al inglés y a otros idiomas. ¡Se aceptan contribuciones!



## Pasantes

Te damos la bienvenida al Programa de pasantías para talentos de código abierto (válido por un largo tiempo). Para obtener más información, haz clic en este [enlace](articles/开源英人才面试.pdf).



## ¿Qué es Xiangshan?

En 2019, con el apoyo de la Academia de Ciencias de China, el Instituto de Tecnología Informática de la Academia de Ciencias de China inició el proyecto de procesador RISC-V de código abierto de alto rendimiento "Xiangshan". , desarrolló el procesador de código abierto de alto rendimiento más alto del mundo. El núcleo del procesador RISC-V de alto rendimiento "Xiangshan" recibió más de 2500 estrellas en GitHub, la plataforma de alojamiento de proyectos de código abierto más grande del mundo, y formó más de 277 ramas (Fork), convirtiéndose en uno de los proyectos de hardware de código abierto más vistos en el mundo. mundo, y ha recibido apoyo activo de empresas nacionales y extranjeras: 16 empresas lanzaron conjuntamente un consorcio de innovación de chips de código abierto para desarrollar conjuntamente "Xiangshan", formar aplicaciones de demostración y acelerar la construcción ecológica RISC-V.

Nuestro objetivo es convertirnos en una plataforma de código abierto para la innovación arquitectónica de cara al mundo, satisfaciendo las necesidades de investigación de arquitectura de la industria, la academia, los entusiastas individuales, etc. Además, esperamos **explorar el proceso de desarrollo ágil de procesadores de alto rendimiento** durante el desarrollo de Xiangshan, establecer un conjunto de procesos de diseño, implementación y verificación de procesadores de alto rendimiento basados ​​en herramientas de código abierto y mejorar significativamente eficiencia del desarrollo del procesador, reducir el umbral para el desarrollo del procesador.

**Xiangshan mantendrá un ciclo de iteración de microestructura y un ciclo de producción de aproximadamente medio año, y continuará llevando a cabo innovaciones en microestructura y practicando métodos de desarrollo ágiles. ** El desarrollo oficial del procesador Xiangshan comenzó en junio de 2020. 1e3fad1 es el valor hash de nuestro [primer envío](https://github.com/OpenXiangShan/XiangShan/commit/1e3fad102a1e42f73b646332d264923bfbe9c77e). Todos los envíos anteriores pertenecen a [NutShell procesador producido por la primera fase del plan One Life One Chip 2019](https://github.com/OSCPU/NutShell). En julio de 2021 se puso en producción la primera versión del procesador Xiangshan (nombre en código Yanqi Lake), alcanzando una frecuencia de 1,3 GHz en un nodo de proceso de 28 nm. En la actualidad (marzo de 2022), el código RTL de la segunda versión de la arquitectura del procesador Xiangshan (nombre en código Nanhu) se ha congelado y el proceso de verificación del diseño del back-end está en marcha, con el objetivo de lograr una frecuencia de 2 GHz en El nodo de proceso de 14 nm. Esperamos mejorar gradualmente el nivel de PPA del procesador Xiangshan a través de la optimización continua y la verificación de cinta, convirtiendo a Xiangshan en un procesador de grado industrial de código abierto y una plataforma de código abierto para la innovación arquitectónica de cara al mundo.

**Xiangshan Processor siempre se adherirá a la estrategia de código abierto y abrirá firmemente todos nuestros códigos de diseño, verificación y herramientas básicas. ** Estamos muy agradecidos por la contribución de la comunidad a Xiangshan. En términos de diseño de hardware, algunos diseños de módulos de procesadores Xiangshan se inspiraron en procesadores de código abierto y documentos públicos, y se hizo referencia a procesadores de código abierto existentes como Rocket-Chip, Berkeley-HardFloat y SiFive Block. -inclusivecache y otros códigos. Basándonos en las herramientas de bus existentes, unidades de punto flotante, cachés del sistema, etc. de la comunidad de código abierto Chisel, modificamos y mejoramos sus funciones y optimizamos indicadores de rendimiento como la frecuencia y el rendimiento. Al mismo tiempo, damos la bienvenida a la comunidad para que desarrolle en base a Xiangshan o utilice el código del proyecto Xiangshan. Entre muchos protocolos de código abierto, elegimos la [Licencia Permisiva Mulan Versión 2](http://license.coscl.org.cn/MulanPSL2/) con la esperanza de (1) mantener la apertura del procesador Xiangshan y La licencia es (2) Con sede en China y de cara al mundo, la licencia relajada de Mulan se expresa tanto en chino como en inglés, y las versiones en chino e inglés tienen el mismo efecto legal.

**El procesador Xiangshan adopta activamente la comunidad de código abierto y da la bienvenida a las contribuciones de la comunidad. ** Hemos visto que algunos proyectos de procesadores RISC-V de código abierto rara vez aceptan envíos de código externo. Entendemos que existen múltiples razones conceptuales y técnicas detrás de este fenómeno, como inquietudes sobre conflictos en el plan de desarrollo y la necesidad de tener en cuenta múltiples aspectos. del diseño del procesador, dificultad para evaluar la calidad de las contribuciones externas, etc. En lo que respecta al proyecto Xiangshan, en términos de conceptos, agradecemos enormemente las contribuciones externas, como el envío de problemas, el envío de requisitos de funciones, el envío de código, etc. Consideraremos y evaluaremos seriamente cada opinión y sugerencia. Por ejemplo, Chisel todavía es nuevo en la industria. Si está más familiarizado con Verilog/SystemVerilog pero desea contribuir con código a Xiangshan, lo invitamos a contribuir a [este repositorio](https://github.com/OpenXiangShan/XS -Verilog-Library) para enviar una solicitud de extracción. A nivel técnico, esperamos explorar un conjunto de procesos y herramientas para evaluar la calidad de los cambios de código y utilizar procesos básicos para determinar si aceptamos el envío de un código. Por ejemplo, esperamos abrir un marco de muestreo de rendimiento más rápido y preciso en el futuro cercano para evaluar los beneficios de rendimiento de un cambio arquitectónico. Aceptaremos esta modificación del código si tiene ciertos beneficios de rendimiento y un buen estilo de codificación. Si resumimos nuestra estrategia de desarrollo en una oración, agradecemos cualquier discusión, presentación de problemas, modificaciones de código, etc. que sean beneficiosas para Xiangshan.

**Además de la exploración de la microestructura, el proyecto Xiangshan también espera explorar y establecer un proceso de desarrollo ágil para procesadores de alto rendimiento. ** El objetivo del procesador Xiangshan es convertirse en una plataforma de código abierto para la innovación arquitectónica de cara al mundo. El establecimiento de capacidades, instalaciones y procesos básicos es la clave para el desarrollo de alta calidad a largo plazo del procesador Xiangshan. Mantendremos Inversión a largo plazo y estable y seguir trabajando duro para construir procesadores. La infraestructura y los procesos básicos del desarrollo ágil. En las primeras etapas del proyecto Xiangshan, adoptamos el marco de desarrollo y verificación del procesador Guokr. Durante el avance del proyecto Xiangshan, realizamos muchas mejoras y agregamos características que incluyen puntos de control de simulación, carga de archivos comprimidos y soporte de verificación de múltiples núcleos. En la actualidad, el entorno de verificación del procesador Xiangshan se ha mejorado enormemente en comparación con el del procesador Guokr, y las ricas herramientas básicas respaldan el ágil proceso de verificación de Xiangshan en este nivel complejo. Además, los frameworks UCB y Chipyard son nuestros modelos a seguir, y hemos hecho referencia a o utilizado muchos proyectos de código abierto iniciados por ellos. Esperamos que a medida que el proyecto Xiangshan avance y se profundice, podamos promover el progreso continuo de la comunidad de código abierto y trabajar con ella para promover el establecimiento de procesos de desarrollo ágiles e infraestructura para procesadores de alto rendimiento.

El desarrollo oficial del procesador Xiangshan comenzó en junio de 2020. Después de menos de dos años de arduo trabajo, hemos logrado la salida de la versión Xiangshan Yanqi Lake y estamos a punto de completar los preparativos para la salida de la versión Nanhu. Desde la perspectiva de los participantes del proyecto Xiangshan, en los últimos dos años, más de 20 profesores y estudiantes de universidades e institutos de investigación como el Instituto de Tecnología Informática de la Academia China de Ciencias, el Laboratorio Pengcheng, la Universidad de Shenzhen, la Universidad de Ciencias de Huazhong y Tecnología y la Universidad de Pekín han participado en el proyecto. Participaron en el proceso de front-end de la versión Yanqi Lake del procesador Xiangshan. Al comienzo del proyecto, la mayoría de los profesores y estudiantes del equipo de Xiangshan no tenían una gran experiencia en el diseño e implementación de procesadores de alto rendimiento, pero después de un año de capacitación en el proyecto Xiangshan, todos establecieron una comprensión preliminar de los procesadores de alto rendimiento. procesadores de rendimiento. En el procesador Xiangshan, el equipo de desarrollo implementó desde cero los códigos clave, incluidos el front-end central, el back-end, el canal de acceso a la memoria, la caché L1 y la compatibilidad con la consistencia de la caché L2. Todo nuestro historial de modificaciones de código está disponible en Visible en Historial de confirmaciones de Git. El proceso de diseño físico del procesador Xiangshan lo completa principalmente nuestro equipo de ingeniería back-end en el Laboratorio de Pengcheng.

**Somos conscientes de que todavía existe una gran brecha entre los procesadores Xiangshan y el nivel general de la industria. Por ejemplo, no hemos hecho un buen trabajo en la selección de soluciones para muchos aspectos técnicos. El objetivo del procesador Xiangshan es convertirse en una plataforma de código abierto para la innovación arquitectónica de cara al mundo. Agradecemos las sugerencias y opiniones de veteranos de la industria, entusiastas de procesadores de alto rendimiento y comunidades de código abierto, siempre que sean beneficiosas para el proyecto del procesador Xiangshan. Lo apoyaremos, lo aceptaremos y lo mejoraremos. Al mismo tiempo, también damos la bienvenida y alentamos a más personas a unirse al desarrollo de los procesadores Xiangshan para promover la innovación continua del proyecto Xiangshan. **




## Informes recomendados sobre Xiangshan

Xiangshan siempre se adhiere al principio de buscar la verdad a partir de los hechos, siempre mantiene la intención original de establecer una plataforma de código abierto y explorar el desarrollo ágil, y no quiere que aparezca ningún contenido que pueda causar malentendidos en su publicidad.

Desde que Xiangshan se lanzó oficialmente en la Cumbre RISC-V de China en junio de 2021, hemos visto muchos informes sobre los procesadores Xiangshan en las redes sociales, medios propios y medios de noticias, algunos de los cuales son "autopublicaciones" del autor. Existe la posibilidad de engañar a los lectores. Para este propósito, hemos creado especialmente una [sección para desmentir rumores](https://github.com/OpenXiangShan/XiangShan-doc/tree/main/clarifications) para aclarar algunos malentendidos generalizados.

Recomendamos los siguientes informes sobre los procesadores Xiangshan, que provienen de nuestros comunicados oficiales y canales de divulgación científica:

- [Introducción al procesador Xiangshan](https://www.zhihu.com/question/466393646/answer/1955410750) escrito por el propio Bao Yungang
- [Artículo](https://mp.weixin.qq.com/s/MAkxKZ1eS4UwBkvgD91Xng) de la cuenta oficial de WeChat de la Alianza del Ecosistema de Instrucción Abierta de China (RISC-V)
- Código fuente del procesador Xiangshan (https://github.com/OpenXiangShan)

Uno de los conceptos centrales de Xiangshan es el código abierto. Con respecto a este tema, recomendamos leer ["Sobre el espíritu del código abierto" del académico Sun Ninghui](https://mp.weixin.qq.com/s/1Irs9a0EKoB7P-J_4ju66A).
