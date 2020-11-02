# 0x3 Dataflow Errors

数据流错误出现在raw SIL中。Swift语义上定义了下列条件为error，因此他们必须被诊断pass检测出来，不能让他们出现在canonical SIL中。

## Definitive Initialization

Swift要求所有局部变量在用之前都进行初始化。
所有struct、enum、class等类型的实例变量都必须先初始化再使用，由构造器进行初始化。


## Unreachable Control Flow

在raw SIL中会输出unreachable terminator来标记错误的控制流，例如non-Void返回值的函数返回了一个值，或者switch没有覆盖所有case等。DCE(dead code elimination，无用代码消除)优化器会把这些不可及的bb代码块给消除掉；另外对于返回uninhabited类型的函数的apply指令，如果是unreachable也可以消除。
在DCE阶段留下来的不可及指令，但没有立即被另一个non-return application处理的话也会认为是一个数据流错误。

```
Uninhabited Types
Never is an uninhabited type, which means that it has no values. Or to put it another way, uninhabited types can’t be constructed.

https://nshipster.com/never/
```