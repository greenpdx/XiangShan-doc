# Introduction to floating-point components

The floating-point components used in the Xiangshan Nanhu version are maintained at [fudian](https://github.com/OpenXiangShan/fudian), which includes the following modules:

* Two-path Floating-point Adder
* Floating-point Multiplier
* Cascade FMA
* Float -> Int Converter
* Int -> Float Converter
* Float -> Float Converter
* Floating-point Divider and SQRT Unit

The FMA operation delay is 5 beats, and the rest of the operations are 3 beats.

## DIV & SQRT

The floating-point division component of Xiangshan uses the SRT16 algorithm like the fixed-point division. The floating-point square root component uses the SRT4 algorithm. Both share the pre-processing and post-processing logic.
