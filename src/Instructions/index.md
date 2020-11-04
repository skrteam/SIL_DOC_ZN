# 0xB Instruction Set



## Reference Counting

这些指令处理堆对象的引用计数。
strong引用类型的值具有被引用堆对象的ownership语义。
在SIL中，retain 和 release 操作必须被显式地写出来。
对值的retain 和 release可能会被移动位置，并且可能会检测它们是否按对匹配。
对值的使用，会维护一个owning retain技计数。

所有引用计数操作都定义为可以正确地处理null引用（strong，unowned,或者weak）。
一个non-null引用必须应用一个明确类型（或自类型）的有效对象。
Address操作数必须是有效的且是non-null.


SIL要求引用计数操作必须是显式的，SIL的类型系统也完整地表达引用的效力（strength）。
```
FYI
reference strength 引用效力
笔者理解为一种语义上引用的关联程度，具体有strong，weak，unowned

```

这有以下的一些好处：
1. 类型安全：原生层是@weak和@unowned的引用，不会在SIL层输出为strong引用。
2. 一致性：当引用在内存中，像copy_addr和destroy_addr这类指令所操作的地址的具体类型，已经包含了正确的语义信息，因此就不需要一些特殊的变体或者flags来描述了。
3. 易工具化：SIL可以直接表示用户对于引用效力的意图，这使得开发一个内存使用分析器（memo profiler）变得简单。原则上，只要对SIL添加一些额外的限制和扩展，甚至可以去掉所有引用计数指令，而基于类型信息实现一个GC（垃圾内存回收器）。