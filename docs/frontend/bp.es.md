# Predicción de rama
<figura rebajada>
 ![bpu](../figs/frontend/bpu.svg){ ancho="800" }
 <figcaption>Diagrama de canalización de BPU</figcaption>
</figura>

Este capítulo describe la arquitectura general de la unidad de predicción de la rama del procesador Xiangshan, cuyo proceso de predicción se muestra en la figura anterior.

<!-- La arquitectura Nanhu adopta una arquitectura de búsqueda de instrucciones que desacopla la predicción de bifurcaciones y la memoria caché de instrucciones. La unidad de predicción de bifurcaciones proporciona solicitudes de búsqueda de instrucciones y las escribe en una cola, que las envía a la unidad de búsqueda de instrucciones y a la memoria caché de instrucciones. -->
La unidad de predicción de rama adopta una arquitectura de predicción híbrida de múltiples niveles, cuyos componentes principales incluyen el [Predictor de siguiente línea](#nlp) (Predictor de siguiente línea, en adelante denominado NLP) y el [Predictor preciso](#apd) (Predictor preciso). Predictor, en adelante APD). Entre ellos, [NLP](#nlp) es un [uBTB](#ubtb) (micro BTB), [APD](#apd) se compone de [FTB](#ftb)[^ftbcite], [TAGE-SC ](# tage-sc), [TIEMPO](#ittage), [RAS](#ras). La PNL proporciona predicciones sin burbujas y los resultados de la predicción se pueden obtener en el siguiente latido de la solicitud de predicción. El retraso de cada componente del APD es de entre 2 y 3 latidos. Entre ellos, el retraso de FTB, TAGE y RAS es de 2 latidos; el retraso de SC e ITTAGE es de 3 latidos. Una predicción pasa por tres etapas de proceso y cada etapa genera nuevo contenido de predicción. Estos predictores distribuidos en diferentes etapas del proceso forman un [predictor primordial](#overriding-predictor) (predictor primordial).

Además de si está desacoplado de la unidad de búsqueda de instrucciones, la mayor diferencia entre el predictor de bifurcación de la arquitectura Nanhu y la arquitectura de la generación anterior (Yanqi Lake) es la definición del [bloque de predicción](#pred-block). En la arquitectura Nanhu, BTB se reemplaza por Fetch Target Buffer. Cada elemento FTB forma un bloque de predicción. No solo predecimos la dirección de inicio del siguiente bloque, sino que también predecimos el punto final del bloque actual. Consulte la sección [FTB](#ftb) para obtener más detalles.

## Módulo de nivel superior (BPU)
BPU (Unidad de predicción de sucursales) es el módulo de nivel superior del predictor de sucursales, que contiene la lógica de predicción de cobertura y la lógica de protocolo de enlace de canalización, así como la gestión del historial de sucursales global.

### Lógica del apretón de manos
Cada etapa del pipeline de la BPU se conectará a [FTQ](ftq.md). Una vez que la primera etapa del pipeline de predicción tenga un resultado de predicción válido, o la etapa del pipeline de predicción posterior produzca un resultado de predicción diferente, se comunicará con [FTQ] (ftq.md) El bit válido de la señal se establecerá en alto.

### Gestión del historial de sucursales a nivel global
La arquitectura de Southlake implementa un historial de sucursales global casi completamente preciso. Esta propiedad está garantizada por los siguientes tres puntos:

- **Actualización especulativa**: cada predicción calculará un nuevo historial global basado en la cantidad de instrucciones de bifurcación condicional y la dirección prevista en el [bloque de predicción](#pred-block) y lo usará en la nueva predicción.
- **Agregar comparación de historial global a la lógica de sobrescritura**: una vez que se actualiza el historial global de la etapa de canalización posterior y el resultado es diferente al de la etapa de canalización anterior (la cantidad de ramas condicionales o el resultado de la ejecución es diferente) , la tubería también se limpiará y reiniciará.
- **Almacenar una copia del historial global después de la predicción**: una vez completada la predicción, el historial global utilizado en la predicción se almacenará en [FTQ](ftq.md), se leerá y se enviará de vuelta a la BPU. Cuando se recupera la predicción errónea

La razón por la que digo "casi" es porque la BPU ignora las instrucciones de bifurcación condicional que nunca saltan. No se registran en el FTB y no se incluyen en el historial de bifurcaciones.

## Predictor de la siguiente fila (PNL)
El siguiente predictor de fila tiene como objetivo proporcionar un flujo de predicción rápido sin cavitación con una pequeña sobrecarga de almacenamiento. Su funcionalidad la proporciona principalmente [uBTB](#ubtb). Para una dirección de inicio PC dada, uBTB realiza una predicción general para un [bloque de predicción](#pred-block) a partir de PC.

###uBTB
Utiliza el historial de ramificaciones y la tabla de almacenamiento de índices XOR de orden inferior de PC. El contenido leído de la tabla proporciona directamente la predicción más concisa, incluida la dirección de inicio del siguiente bloque de predicción "nextAddr", si este bloque de predicción tiene una ramificación salto de instrucción y si se produce el salto de instrucción de bifurcación. Se toma la instrucción de salto. Si se toma un salto, la instrucción de salto se desplaza desde la dirección de inicio mediante cfiOffset . Además, se proporciona información relevante sobre la instrucción de bifurcación para actualizar el historial de bifurcación, incluido si el salto de bifurcación condicional se realiza en Br y la cantidad de instrucciones de bifurcación en el bloque, brNumOH.

Hemos abandonado la práctica de coincidencia de etiquetas, lo que trae un problema. En el caso de coincidencia de etiquetas, si un [bloque de predicción](#pred-block) no coincide, el PC del siguiente bloque de predicción se establecerá en el PC actual más el [ancho de predicción](#pred-width). Para evitar desperdicios, si no hay ninguna instrucción de bifurcación en el bloque de predicción, no se escribe ningún uBTB durante el entrenamiento. Bajo esta premisa, si no hay un mecanismo de coincidencia de etiquetas, es fácil predecir un bloque predicho sin instrucciones de bifurcación (que no impacta bajo el mecanismo de coincidencia de etiquetas) como otro bloque de salto. Para abordar esta situación, introdujimos otro mecanismo de predicción para predecir si puede haber una instrucción de bifurcación válida para la PC actual. La estructura de almacenamiento de este mecanismo de predicción está indexada directamente por la PC de obtención de instrucciones, y su resultado de consulta indica si se ha escrito la PC de obtención de instrucciones. Cuando indica que no se ha escrito el PC, se predice que el próximo PC será el PC actual más el ancho previsto.

## Predictor Preciso (APD)
Para mejorar la precisión general de la predicción y reducir el vaciado de la tubería causado por errores de predicción, la arquitectura Nanhu implementa un mecanismo de predicción con mayor latencia y mayor precisión.

El predictor preciso incluye el búfer de destino de instrucción [FTB](#ftb), el predictor de dirección de rama condicional [TAGE-SC](#tage-sc), el predictor de salto indirecto [ITTAGE](#ittage) y la pila de dirección de retorno. [RAS](#ras).


Por favor, lea
FTB es el núcleo de APD. Las predicciones realizadas por los demás componentes de predicción de APD se basan en la información proporcionada por FTB. Además de proporcionar información sobre las instrucciones de bifurcación dentro del [bloque de predicción](#pred-block), FTB también proporciona la dirección final del bloque de predicción. Para FTB, la estrategia de generación de artículos FTB es crucial. La arquitectura Nanhu se basa en el artículo original [^ftbcite] y combina las ideas de este artículo [^ftbcitequalcomm] para formar la estrategia actual. La dirección inicial del elemento FTB es *`start`* y la dirección final es *` fin `*, las estrategias específicas son las siguientes:

- **Los elementos FTB se indexan mediante *`start`*, que se genera en el proceso de predicción. En la práctica, *`start`* sigue básicamente uno de los siguientes principios:**
 - *`start`* es el *`end`* del bloque de predicción anterior
 - *`start`* es la dirección de destino de las redirecciones desde fuera de la BPU;
- **Se pueden registrar como máximo dos instrucciones de bifurcación en el elemento FTB, y la primera debe ser una bifurcación condicional;**
!!! nota en línea final "Nota"
 Bajo esta estrategia de entrenamiento, la misma instrucción de rama puede existir en múltiples entradas FTB.
- ***fin* debe satisfacer una de las siguientes tres condiciones:**
 - *`end`* - *`start`* = ancho previsto
 - *`end`* es el PC de la tercera instrucción de rama en el rango de ancho de predicción que comienza desde *`start`*
 - *`end`* es el PC de la siguiente instrucción de una rama incondicional, y está dentro del ancho de predicción de *`start`*



Al igual que en la implementación del artículo [^ftbcite], solo almacenamos los bits bajos de la dirección final, y los bits altos se concatenan con los bits altos de la dirección inicial. De manera similar a AMD[^amd], también registramos el bit "saltar siempre" para las instrucciones de bifurcación condicional en los elementos [FTB](#ftb), que se establece la primera vez que se encuentra la bifurcación condicional. 1, cuando su valor es 1 , siempre se predice que la dirección de la rama condicional saltará, y su resultado no se usa para entrenar al predictor de dirección de la rama condicional; cuando la rama condicional encuentra un resultado de ejecución de no saltar, establezca este bit en 0, después de lo cual su dirección es predicho por el predictor de dirección de rama condicional.

### DÍA-SC
TAGE-SC es el predictor principal de las ramas condicionales en la arquitectura Nanhu. Su lógica general se hereda de TAGE-SC-L de la arquitectura Yanqi Lake de la generación anterior. En la implementación actual, el retraso de TAGE es de 2 latidos y el retraso de SC es de 3 latidos.
!!! nota "¿Por qué no hay un predictor de bucle?"
 La arquitectura de Southlake elimina el predictor de bucle porque la forma en que se definen los elementos FTB en la arquitectura actual hará que exista una instrucción de bifurcación condicional en varios elementos FTB al mismo tiempo, lo que causará problemas para registrar con precisión la cantidad de bucles de una bifurcación de bucle. instrucción. Es difícil. En la arquitectura de Yanqi Lake, se realiza una predicción para cada instrucción y una instrucción de bifurcación solo aparece una vez en el BTB, por lo que no existe ese problema.
<figura rebajada>
 ![bpu](../figs/frontend/tage.png){ ancho="600px" }
 <figcaption>Diagrama lógico básico de TAGE</figcaption>
</figura>
TAGE explota múltiples tablas de predicción con longitudes de historial que aumentan exponencialmente para extraer información del historial de ramas extremadamente largas. Su lógica básica se muestra en la figura anterior. Consiste en una tabla de predicción base y varias tablas de historial. La tabla de predicción base está indexada por PC, mientras que la tabla de historial está indexada por PC y el resultado de plegar una cierta longitud de historial de rama. La longitud del historial de rama utilizado por diferentes historiales Las tablas son geométricamente proporcionales. Relación numérica. Durante la predicción, la etiqueta se calcula mediante la operación XOR del PC y otro resultado de plegado del historial de la rama correspondiente a cada tabla de historial, y se combina con la etiqueta leída de la tabla. Si la coincidencia es exitosa, la tabla coincide. El resultado final depende del resultado de la tabla de predicción con el historial de aciertos más largo. En la arquitectura de Southlake, se pueden predecir como máximo 2 instrucciones de bifurcación condicional simultáneamente en cada predicción. Al acceder a cada tabla de historial de TAGE, la dirección de inicio del bloque de predicción se utiliza como PC, y se extraen dos resultados de predicción al mismo tiempo, y el historial de la rama que utilizan también es el mismo.
!!! nota "Razón por la cual dos ramas utilizan la misma predicción histórica"
 En teoría, cada rama debería utilizar el historial de ramas más reciente, porque en general, los bits más cercanos a la rama actual en la secuencia del historial de ramas tienen una mayor probabilidad de afectar el resultado de la rama actual. Para la segunda rama, el historial de la última rama debe contener los resultados de la primera rama. Aquí, la predicción de las dos ramas condicionales puede usar el mismo historial de ramas porque si se necesita el resultado del salto de la segunda rama, entonces la primera rama no debe saltar, por lo que la última rama para la segunda rama El bit de historial debe ser 0. Esto es seguro, por lo que no introduce mucha información. Los resultados de la prueba también muestran que la ganancia de precisión al utilizar diferentes predicciones del historial de ramas para los dos es casi insignificante, pero introduce una lógica compleja.

TAGE también tiene una lógica de [predicción alternativa](#alt_pred). Implementamos el registro `USE_ALT_ON_NA` basado en el diseño de L-TAGE[^ltage] para decidir dinámicamente si se debe utilizar la predicción alternativa cuando la confianza en la coincidencia histórica más larga El resultado es insuficiente. En la implementación, debido a consideraciones de tiempo, los resultados de la tabla de predicción base siempre se utilizan como predicciones alternativas, lo que produce poca pérdida de precisión.

La entrada de la tabla TAGE contiene un campo "útil". Si su valor no es 0, significa que el elemento es útil y el algoritmo de asignación no lo asignará como elemento vacío durante el entrenamiento. Durante el entrenamiento, utilizamos un contador de saturación para monitorear dinámicamente la cantidad de asignaciones exitosas/fallidas. Cuando la cantidad de asignaciones fallidas es lo suficientemente alta y el contador alcanza la saturación, borramos todos los campos "útiles" y los ponemos a cero.

SC es un corrector estadístico que revierte el resultado de la predicción final cuando cree que TAGE tiene una gran probabilidad de predicción errónea. Su implementación se basa en la estructura del predictor O-GEHL [^o_gehl] y es esencialmente una variante de la lógica de predicción del perceptrón [^perceptron].

!!! nota "Historial de pliegues"
 Cada tabla de historial del predictor de clase TAGE tiene una longitud de historial específica. Para indexar la tabla de historial después de la operación XOR con el PC, es necesario dividir una secuencia de historial de ramificación muy larga en muchos segmentos y luego realizar la operación XOR en conjunto. La longitud de cada segmento es generalmente igual al logaritmo de la profundidad de la tabla histórica. Dado que la cantidad de XOR generalmente es grande, para evitar la demora de los XOR de múltiples niveles en la ruta de predicción, almacenamos directamente el historial plegado. Dado que los historiales de diferentes longitudes se pliegan de diferentes maneras, la cantidad de historiales plegados necesarios es igual a la cantidad de tuplas (longitud del historial, longitud plegada) después de la deduplicación. Al actualizar un bit del historial, solo es necesario realizar una operación XOR entre el bit más antiguo antes de plegar y el bit más nuevo en la posición correspondiente, y luego realizar una operación de desplazamiento.




### ITALIA
La instrucción `jalr` en el conjunto de instrucciones RISC-V permite especificar la dirección de destino de una instrucción de salto incondicional agregando un valor inmediato a un valor de registro. A diferencia de la instrucción `jal`, que codifica directamente el desplazamiento de salto en la instrucción, la dirección de salto de `jalr` debe obtenerse indirectamente a través del acceso al registro, por lo que se denomina instrucción de salto indirecto. Dado que el valor del registro es variable, la dirección de salto de la misma instrucción `jalr` puede ser muy diferente, por lo que el mecanismo FTB de registro de direcciones fijas hace difícil predecir con precisión la dirección de destino de dichas instrucciones. En el procesador Xiangshan, la instrucción `jalr`
