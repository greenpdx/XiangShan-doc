# Introducción a los componentes de punto flotante

Los componentes de punto flotante utilizados por la versión Xiangshan Nanhu se mantienen en [fudian](https://github.com/OpenXiangShan/fudian), que contiene los siguientes módulos:

* Sumador de punto flotante de dos rutas
* Multiplicador de punto flotante
* Cascada FMA
* Convertidor de flotante -> entero
* Convertidor Int -> Flotante
* Convertidor de flotante -> flotante
* Divisor de punto flotante y unidad SQRT

El retraso de la operación FMA es de 5 tiempos y las otras operaciones tienen un retraso de 3 tiempos.

## DIV y RAIZ CUADRADA

La unidad de división de punto flotante de Xiangshan utiliza el algoritmo SRT16, al igual que la división de punto fijo. El componente de raíz cuadrada de punto flotante utiliza el algoritmo SRT4. Ambos comparten lógica de preprocesamiento y posprocesamiento.
