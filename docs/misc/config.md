Xiangshan parameter system description
===================

Xiangshan's parameter system uses the parameter environment Context-Dependent Evironments, which corresponds to the api-config-chipsalliance submodule in the repository. For more detailed instructions, please refer to [the document of this repository](https://github.com/chipsalliance/api-config-chipsalliance).

To modify Xiangshan's configuration, please refer to the processing method of `MinimalConfig` in [Configs.scala](https://github.com/OpenXiangShan/XiangShan/blob/master/src/main/scala/top/Configs.scala). When building your own Config, you only need to write the parts that are different from the default configuration in the Config.

After defining your own Config in a similar way, you can use the `CONFIG` parameter to determine which configuration to use when generating Verilog/simulation. For example: `make emu CONFIG=MinimalConfig`.

The default parameters of Xiangshan are in [Parameters.scala](https://github.com/OpenXiangShan/XiangShan/blob/master/src/main/scala/xiangshan/Parameters.scala), and it is generally not recommended to modify them.

# Example

For example, if you want to modify the ROB and IBuffer size of Xiangshan, you can create a new `MyConfig` [here](https://github.com/OpenXiangShan/XiangShan/blob/master/src/main/scala/top/Configs.scala). `MyConfig` will inherit all the default parameters of Xiangshan. Next, modify the parameters of ROB and IBuffer in `MyConfig`:

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

Then, when generating the simulator and verilog code, add the option `CONFIG=MyConfig` in the `make` command to replace the default `DefaultConfig` with `MyConfig`.

