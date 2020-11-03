# 0x5 Runtime Failure

某些操作（例如失败的无条件检查转换（checked conversions）或内置的Builtin.trap编译器）会导致运行时失败，从而无条件地终止当前actor。如果可以证明将发生或确实发生了运行时failure，则可以对运行时failure进行重新排序，只要它们相对于actor或整个程序的外部操作保持良好的顺序即可。例如，启用对int算术的溢出检查后，一个简单的for循环会从一个或多个数组中读取输入并将输出写入到另一个数组中，这些数组都是当前actor本地的变量，这可能会导致更新操作中的运行时失败：

```
// Given unknown start and end values, this loop may overflow
for var i = unknownStartValue; i != unknownEndValue; ++i {
  ...
}
```

允许将溢出检查和相关的运行时failure从循环本身中提升出来，并在进入循环之前检查循环的边界，只要循环体在当前actor之外没有可观察到的外部作用即可。

