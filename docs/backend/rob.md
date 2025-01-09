# Buffer de reordenamiento (ROB)

En un procesador fuera de servicio, la función del ROB es secuenciar instrucciones para que se puedan preservar los resultados de la ejecución normal del programa. Según el proceso de ejecución de una instrucción, ROB afectará los procesos de despacho, escritura diferida y envío de la instrucción en secuencia, y la instrucción puede eliminarse en cualquier momento.

## Despacho

En la fase de Envío, a la instrucción se le asignará una entrada ROB, y cierta información que debe guardarse en `RobCommitInfo` se almacenará en el ROB, como la información de cambio de nombre de la instrucción, la información de tipo, el puntero ftq correspondiente, etc. El ancho de la cola ROB es consistente con el ancho renombrado.

Después de que la instrucción ingresa al ROB, se actualizarán algunos bits de estado, como `válido`, `writebacked`, `interrupt_safe` (para simplificar el diseño, la información de si la instrucción de acceso a memoria es MMIO no se pasará al ROB El acceso a la memoria ocurre antes del envío, por lo que actualmente simplificamos el diseño para evitar que las instrucciones de acceso a la memoria activen interrupciones), etc.

## Escribir de nuevo

Después de ejecutar la instrucción, se notificará al ROB que se ha completado la operación correspondiente y el ROB establecerá el indicador "writebacked" en "true".

## entregar

En cada ciclo de reloj, ROB verificará si las instrucciones al principio de la cola se pueden enviar normalmente y enviará tantas instrucciones como sea posible a través de la interfaz `io.commits`.

Para las instrucciones con excepciones, se bloqueará su envío y la información de la excepción se enviará a través de la interfaz `io.exception`.

## Cancelar y revertir

Durante la ejecución de una instrucción, si ocurre un error de predicción de bifurcación, una violación de acceso a la memoria, etc., es posible que sea necesario eliminar la instrucción y las instrucciones subsiguientes. En este caso, ROB recibirá la información de cancelación a través del puerto `io.redirect` y utilizará la información de cancelación para determinar qué parte de la instrucción debe cancelarse. En el caso de las instrucciones canceladas, ROB utilizará el mecanismo de reversión para restaurar el cambio de nombre y otra información a través del puerto `io.commits`. En este momento, `io.commits.isWalk` se establecerá en `true`.
