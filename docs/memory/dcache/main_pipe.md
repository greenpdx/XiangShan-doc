# Tubería principal

La arquitectura NanHu utiliza la canalización principal (canalización principal de dcache) para gestionar el almacenamiento, la prueba, las instrucciones atómicas y las operaciones de reemplazo. La canalización principal es responsable de la ejecución de todas las instrucciones que deben competir por la cola de reescritura para iniciar solicitudes/reescrituras. datos a la caché de nivel inferior.

!!! información
 En comparación con la arquitectura de Yanqi Lake de la generación anterior, la arquitectura de Nanhu separa la operación de recarga de la tubería principal en una tubería separada. Otras operaciones no están separadas de la tubería principal porque el costo de lidiar con conflictos/contención de recursos entre múltiples tuberías excede los beneficios de rendimiento. de división de tuberías.

## Funciones de MainPipe en cada nivel del pipeline

### Etapa 0

* Arbitrar las solicitudes entrantes de la tubería principal y seleccionar la que tenga la mayor prioridad
* Determinar si los recursos necesarios para la solicitud están disponibles en función de la información de la solicitud.
* Etiqueta de problema, solicitud de lectura meta

Etapa 1

* Obtener el resultado de la solicitud de lectura de etiquetas y metadatos
* Realizar una comprobación de coincidencia de etiquetas para determinar si es un éxito
* Si se requiere reemplazo, obtenga el resultado de selección de reemplazo proporcionado por PLRU
* Realizar una verificación de permisos en función de los metadatos de lectura.
* Determinar de antemano si se requiere acceso a la cola de espera

Etapa 2

* Obtener el resultado de la lectura de datos y combinarlo con los datos que se van a escribir.
* Si falla, intente escribir esta información de solicitud en la cola de fallas

<!-- El tiempo de escritura de la cola de fallas es muy ajustado, esta canalización permite que la escritura de la cola de fallas comience lo antes posible para reducir su impacto en el tiempo-->

Etapa 3

* Actualizar meta, datos, etiquetas según el resultado de la operación.
* Si la instrucción necesita acceder o escribir datos en el nivel de caché inferior, se genera una solicitud de acceso a la cola de escritura diferida en este nivel y se intenta escribir en la cola de escritura diferida.
* Cuando la operación de liberación surte efecto y actualiza los metadatos, se envía una señal de liberación a la cola de carga en este nivel para determinar la violación.
* Soporte especial para instrucciones atómicas
 * La instrucción AMO permanece en esta etapa durante dos tiempos, primero completando la operación de la instrucción AMO en esta etapa de la cadena de bloques y luego escribiendo el resultado nuevamente en el caché de datos.
 * Los comandos LR/SC establecerán/verificarán su conjunto de reservas aquí
 cola

<!-- ## Flujo de ejecución de instrucciones -->

<!-- !!! todo -->
<!-- Los diferentes tipos de solicitudes (almacenar/probar/reemplazar/atomizar) explican el flujo de ejecución de instrucciones en mainpipe -->

## Contención y bloqueo de MainPipe

La contención del Oleoducto Principal tiene las siguientes prioridades:

1. solicitud de sonda
1. reemplazar_req
1. tienda_req
1. requerimiento atómico

Una solicitud se acepta solo si todos los recursos que solicita están listos, no hay ningún *conflicto de conjunto* y no hay ninguna solicitud con una prioridad mayor que la suya. Las solicitudes de escritura desde los buffers de almacenamiento comprometidos tienen una lógica de verificación independiente.

<!-- ### establecer prevención de conflictos -->

<!-- *establecer conflicto* TODO -->


<!-- ## Manejo de errores en MainPipe -->

<!-- !!! todo -->
