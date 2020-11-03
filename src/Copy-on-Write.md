# 0xA Copy-on-Write Representation

Copy-on-Write (COW)数据结构是通过对象引用来实现的，在数据修改时copy一份数据，并建立新的引用。


SIL中典型的COW修改操作序列如下：
```
 (%uniq, %buffer) = begin_cow_mutation %immutable_buffer : $BufferClass
  cond_br %uniq, bb_uniq, bb_not_unique
bb_uniq:
  br bb_mutate(%buffer : $BufferClass)
bb_not_unique:
  %copied_buffer = apply %copy_buffer_function(%buffer) : ...
  br bb_mutate(%copied_buffer : $BufferClass)
bb_mutate(%mutable_buffer : $BufferClass):
  %field = ref_element_addr %mutable_buffer : $BufferClass, #BufferClass.Field
  store %value to %field : $ValueType
  %new_immutable_buffer = end_cow_mutation %buffer : $BufferClass
```

从COW数据结构load操作如下：

```
%field1 = ref_element_addr [immutable] %immutable_buffer : $BufferClass, #BufferClass.Field
%value1 = load %field1 : $*FieldType
...
%field2 = ref_element_addr [immutable] %immutable_buffer : $BufferClass, #BufferClass.Field
%value2 = load %field2 : $*FieldType
```

immutable属性意味着用ref_element_addr和ref_tail_addr指令loading值，如果指令的操作数相同，那么可以认为是等效的。换句话说，只要作为操作数的buffer引用相同，则可以保证buffer的属性在两个ref_element / tail_addr [immutable]之间不会发生修改。甚至该buffer如果逃逸（escape）到其他未知函数，以上也成立。


在上面的示例中，因为两个ref_element_addr指令的操作数都是相同的％immutable_buffer，％value2和％value1相等。从概念上讲，COW buffer对象的内容可以看作是与缓buffer引用相同的静态（不可变）SSA值的一部分。


通过begin_cow_mutation和end_cow_mutation指令可以将COW值的生存期严格分为可变和不可变区域：

```
%b1 = alloc_ref $BufferClass
// The buffer %b1 is mutable
%b2 = end_cow_mutation %b1 : $BufferClass
// The buffer %b2 is immutable
(%u1, %b3) = begin_cow_mutation %b1 : $BufferClass
// The buffer %b3 is mutable
%b4 = end_cow_mutation %b3 : $BufferClass
// The buffer %b4 is immutable
...
```


begin_cow_mutation和end_cow_mutation都会consume其操作数，并把新的buffer作为一个@owned值进行返回。 begin_cow_mutation将编译为唯一性检查代码，而end_cow_mutation将编译为no-op（无操作空代码）。

尽管返回的buffer引用的物理指针值与操作数相同，但在SIL中生成新的buffer引用还是很重要的。它防止优化器将buffer访问从mutable的区域移动到immutable的区域，反之亦然。

因为buffer内容在概念上是buffer引用SSA值的一部分，所以每次更改buffer内容时都必须有一个新的buffer引用。

为了说明这一点，让我们看一个示例，其中一个COW值在一个循环中被更改。与标量SSA值一样，对COW buffer进行更改也将在循环头块中强制使用phi参数（为简单起见，未显示用于复制非唯一缓冲区的代码）：

```
Phi node 
LLVM 中关于SSA的数据结构

The 'phi' instruction is used to implement the φ node in the SSA graph representing the function.

ref： http://llvm.org/docs/LangRef.html#phi-instruction
```


```
header_block(%b_phi : $BufferClass):
  (%u, %b_mutate) = begin_cow_mutation %b_phi : $BufferClass
  // Store something to %b_mutate
  %b_immutable = end_cow_mutation %b_mutate : $BufferClass
  cond_br %loop_cond, exit_block, backedge_block
backedge_block:
  br header_block(b_immutable : $BufferClass)
exit_block:
```

两条相邻的begin_cow_mutation和end_cow_mutation指令不必在同一函数中。