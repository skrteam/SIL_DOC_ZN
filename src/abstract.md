# 0x0 摘要
---------

SIL是Swift的中间语言，提供了和Swift编程语言实现相关的高层的语义信息，其形式遵循了SSA（Static-Single-Assignemt）标准。它支持以下的用例：

- 提供高层的guaranteed optimization，提供可预期的运行时优化基线，并在诊断分析中提供可预期的行为。
- 提供诊断分析中的数据流分析通道处理器（Passes），以确保代码符合Swift的强制规范。包括：变量的初始化检查，构造器缺失检查，代码可及性检查，switch分支的条件覆盖分析等。
- 提供高层的编译优化通道，包括：ARC优化，动态方法去虚拟化，闭包内联化，使用堆内存提升为使用栈内存，使用栈内存提升为使用SSA寄存器，聚合内存标量替换（scalar replacement of aggreates，把聚合的大块内存申请分割为多个小块内存申请），泛型函数实例化等。
- 为Swift库提供一种稳定的分发格式，引用端可以使用内联、泛型等特性进行编译优化。

相比LLVM的中间语言，通常SIL是“体系结构无关”的，即它不关心具体输出的目标代码运行在什么架构平台上，但它也可以和LLVM一样做到“target-specific”。

了解更多关于开发SIL和SIL通道处理器的信息，参见《SILProgrammersManual》

## GOSSARY
guaranteed optimization |        | 一种基于证据的可被检验的编译器优化技术
pass ｜ 通道处理器 ｜ 可以理解为单向的通道处理器
scalar replacement of aggreates｜ 聚合内存标量替换 ｜一种优化，把聚合的大块内存申请分割为多个小块内存申请
target-specific ｜ 指定目标产物｜ 指定输出的产物为某平台或体系结构所支持的格式