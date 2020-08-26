# SIL Stages

```
decl ::= sil-stage-decl
sil-stage-decl ::= 'sil_stage' sil-stage

// 同一SIL文件只描述一种Stage
sil-stage ::= 'raw'
sil-stage ::= 'canonical'
```

对于不同的编译阶段，对应了不同的SIL的具体形式，或者说变体的版本。
主要有两个阶段和形式，同一个SIL文件只存在一种形式：

- **Raw SIL** 这种形式是由SILGen生成，但还未通过guaranteed optimizations和diagnostic passes处理之前的SIL代码。 这时候的Raw SIL可能还未完全构造成SSA图的形式，其数据流可能存在errors。一些指令也是临时的形式，而不是正式canonical的形式，例如，non-address-only类型（见下文）的assign和destroy_addr指令等。


- **Canonical SIL** 是经过guaranteed optimizations和diagnostic passes处理之后的代码，此时数据流中的错误已经消除，部分临时的指令也经过处理变成了正式指令形式。基于Canonical SIL 可以进行性能优化并生成最终的目标文件。另外，其也可以作为组件模块分发的一种格式。



