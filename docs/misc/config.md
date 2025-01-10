Descripción del sistema de parámetros de Xiangshan
==================

El sistema de parámetros de Xiangshan utiliza el parámetro Entornos dependientes del contexto, que corresponde al submódulo api-config-chipsalliance en el repositorio. Para obtener instrucciones más detalladas, consulte la [documentación de este repositorio](https://github.com/ chipsalliance/api-config-chipsalliance).

Para modificar la configuración de Xiangshan, consulte el método de manejo de `MinimalConfig` en [Configs.scala](https://github.com/OpenXiangShan/XiangShan/blob/master/src/main/scala/top/Configs.scala). Cuando creas tu propia configuración, solo necesitas escribir las partes que son diferentes de la configuración predeterminada en la configuración.

Defina su propia configuración de manera similar y luego use el parámetro `CONFIG` para determinar qué configuración usar al generar Verilog/simulación. Por ejemplo: `make emu CONFIG=MinimalConfig`.

Los parámetros predeterminados de Xiangshan están en [Parameters.scala](https://github.com/OpenXiangShan/XiangShan/blob/master/src/main/scala/xiangshan/Parameters.scala), y generalmente no se recomienda modificarlos. a ellos.


# Ejemplo

Por ejemplo, si desea modificar los tamaños de ROB e IBuffer de Xiangshan, puede hacerlo aquí: Cree un nuevo `MyConfig`. `MyConfig` heredará todos los parámetros predeterminados de Xiangshan. A continuación, modifique los parámetros de ROB e IBuffer en ` MiConfig`:

```scala
class MyConfig(n: Int = 1) extends Config(
  new DefaultConfig(n).alter((site, here, up) => {
    case SoCParamsKey => up(SoCParamsKey).copy(
      cores = up(SoCParamsKey).cores.map(_.copy(
        RobSize = 32,
        IBufSize = 16
    )))
  })
)
```

Luego, al generar el simulador y el código verilog, agregue la opción `CONFIG=MyConfig` en el comando `make` para reemplazar el `DefaultConfig` predeterminado por `MyConfig`.
