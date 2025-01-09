# Out-of-order scheduler

In the Xiangshan processor, the `Scheduler` module is the core of out-of-order scheduling, which maintains modules such as the reservation station `ReservationStation`, the physical register file `Regfile`, and the physical register status `BusyTable`. The top level of `Scheduler` has almost no logic, and most of it is the connection between modules.
