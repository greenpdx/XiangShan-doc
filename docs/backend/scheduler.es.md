# Programador fuera de servicio

En el procesador Xiangshan, el módulo `Scheduler` es el núcleo de la programación fuera de orden, que mantiene módulos como la estación de reserva `ReservationStation`, el archivo de registro físico `Regfile` y el estado de registro físico `BusyTable`. Casi no hay lógica en el nivel superior del «Programador», la mayor parte es cableado entre módulos.
