# 0x1 SIL和Swift编译器
--------------------

在高层设计中，swift编译器遵循严格的管道设计架构：

- 源码输入 *Parser* 模块进行解析处理，输出构造好的AST抽象语法树
- AST输入 *Sema* 模块进行语义分析和类型检查，进一步标注更丰富的类型信息，输出AST‘
- AST’输入 *SILGen* 模块生成 *raw SIL*
- *raw SIL* 会输入到一系列 *可检验的优化处理器(Guaranteed Optimization Passes)* 和 *诊断处理器*中进行优化，并输出可能的警告和错误，并输出 *cannonical SIL* 。 注意，以上管道处理器是编译过程的必选项，必然运行
- (可选) *cannonical SIL* 输入到通用管道处理器中进行最终的性能优化。这个过程是可选的，可以开关或者指定特定的优化等级
- 优化后的*cannonical SIL* 输入 *IRGen* 模块，降级转化为LLVM的IR语言
- (可选) 降级后LLVM IR输入到的LLVM后端编译器进行优化处理，并最终生成二进制目标文件。


与SIL处理相关的特定阶段如下：


## SILGen


SILGen输入已确定类型的AST',通过遍历AST'，生成 *raw SIL* 。
 *raw SIL* 具有以下特点：

 - 变量的读写是可变内存地址，没有使用严格的SSA形式。这有点像最早Clang等前端编译器输出的alloca-heavy(大量使用alloc)的LLVM IR。 另一方面，Swift把变量包裹在一个 "Boxes"的容器里，对这个通用结构进行引用计数管理，同时也作为闭包截获外部变量的容器。
 - 数据流的必要检查不在这个阶段执行。包括声明式赋值，函数返回，switch条件覆盖率等。
 - 强行函数内联(transparent function)优化不在这个阶段


## Guaranteed Optimization and Diagnostic Passes


在SILGen生成 *raw sil* ，输入到必选的优化管道处理器中，随后输入诊断管道处理器中。如果源码不变，我们不希望以为编译器的升级或改动导致在该阶段输出的错误、警告等诊断信息跟着变化，因此这要求这些争端处理器都设计得足够简单，并且可以预测输出结果。

- ** 强行内联化(Mandatory inlining)** 也称为透明函数
- **内存提升（Memory Promotion）** 是指把内存使用的方式优化提升为更高效的方式，该处理有两个阶段：第一阶段是把``alloc_box`` 的堆内存申请指令提升为 ``alloc_stack`` 栈内存； 第二阶段是把地址非暴露方式的``alloc_stack`` 栈内存方式提升为使用 SSA寄存器的方式。
- **常量折叠（Constant Folding）** 把常量或表达式不断地简化； **Constant propagation 常量传递** 把常数的值替换到表达式中，从而使得代码可以进一步优化。
- **返回分析** 确认代码的每一段逻辑分支都能符合函数原型的定义。对于有返回类型的函数，确保每个分支的执行末端都会有正确类型的返回；对于不返回的函数，则确保每个分支都不会返回；否则，报错。
- **临界拆分** 是指把多个BB(BasicBlock)拆开，主要针对的是“非cond_branch”的terminator的BB


```
Constant folding 
Constant propagation 
ref: https://zh.wikipedia.org/wiki/%E5%B8%B8%E6%95%B8%E6%8A%98%E7%96%8A


BasicBlock
由顺序执行的指令组成，是串型且不分叉的执行过程，是SIL程序的执行片段和组织单位。
可以理解为拆分开的子程序


terminator 
是SIL一类特殊的指令，是每一个BB的最后一条指令，表明了这段BB该如何结束。
cond_branch是具体的一个terminator指令，指条件分支结束指令，会接受一个参数，根据参数决定控制流的跳转逻辑。
```


如果以上的诊断都通过了，那么会输出Canonical SIL代码。

TODO：
- 泛型特化
- 基础ARC优化，指为了达到可接受的性能而进行的最基本的，而且是强制的ARC优化，即使在-Onone模式也会执行


## General Optimization Passes

SIL可以提供更多和语言特性相关的类型信息，从而可以在高层级(high-level)进行优化。而LLVM IR不适合做这类优化，因为它是通用的，和高级语言特性无关的。这也是SIL存在的意义。

- **泛型特化**：去泛化，去抽象化，生成和具体类型绑定的代码
- **Witness 和 VTable 反虚拟化技术**
- **性能向强制内联化**：为了提升执行效率，强制把可内联化的函数内联化。
- **引用计数优化**
- **内存提升/内存优化**
- **High-level domain specific optimizations** 配合标准库进行的高级优化，比如对Swift容器类型（Array 、String等）进行高层级的优化，具体见
`HighLevelSILOptimizations <HighLevelSILOptimizations.rst>`_



