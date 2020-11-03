# 0x9 Value Dependence

总的来说，分析器可以假定独立的值独立保证其值的有效性。
比如以下的例子，一个类方法返回一个类引用：

```
bb0(%0 : $MyClass):
  %1 = class_method %0 : $MyClass, #MyClass.foo
  %2 = apply %1(%0) : $@convention(method) (@guaranteed MyClass) -> @owned MyOtherClass
  // use of %2 goes here; no use of %1
  strong_release %2 : $MyOtherClass
  strong_release %1 : $MyClass
```

优化器可以把strong_release %1这一行移动到函数apply之后，因为编译器认为%2是独立管理的值，并且Swift一般是允许调整解构顺序的。

有一些指令的执行会创建操作数之间的依赖关系。
比如，如果获取一个结合类型的对象内部元素的指针，而这个集合对象在指针放阿文钱释放里，那么就会出现野指针。
因此，也就需要 Value-Dependence的概念。所以，对解构顺序进行优化重排的化，需要特别注意值之间的依赖关系。

在以下场景，我们可以说值%1依赖于值%0:
- 对于以下指令，%0是第一个操作数，%1是操作结果
  - ref_element_addr
  - struct_element_addr
  - tuple_element_addr
  - unchecked_take_enum_data_addr
  - pointer_to_address
  - address_to_pointer
  - index_addr
  - index_raw_pointer
  - possibly some other conversions
- mark_dependence指令，%1是操作结果，%0是任意操作数
- box_alloc指令，%1是address，%0是ref
- struct,enum,tuple指令，%1是结果，%0是操作数
- tuple_extract, struct_extract, unchecked_enum_data, select_enum, or select_enum_addr指令，%0是聚合对象，%1是提取出来的子对象
- select_value指令，%1是结果，%0是某个case（switch case）
- 对于一个BasicBlock，%1是形参，%0是实参
- 对于load指令，%0是地址操作数，%1是结果。然而如果%1一旦被managed，这种依赖关系就切断了，（因为%1可以独立管理了）。例如retain %1。
- 依赖的传递性。但是这个性质不能引用到struct、tuple、enum的不同子对象上。


另外，需要注意：
- 对内存的依赖关系进行追踪分析不是必要的；
- 对不透明机制的代码进行“透明的”依赖关系分析也不是必要的。比如一个方法返回一个执行类属性的unsafe pointer。
- 对函数内部SIL指令的依赖关系分析是必要的。
- 对于SILGen(用mark_dependence指令是否合理)和用户（是否合理地使用了unsafe语言提供的一些属性或者库的特性）是否生成了合适的依赖关系，需要特别注意。

只有特定的SIL Value可以携带值依赖关系的信息：
- SIL address types
- unmanaged pointer types:
  - @sil_unmanaged types
  - Builtin.RawPointer
- 包含像UnsafePointer类型元素的聚合容器；有可能形成递归依赖关系
- non-trivial types (但也可被独立managed)

依据以上规则，把pointer转成Int，就丢失了携带的依赖关系。但是如果只是为了读Int值的话，也就不用把携带的依赖关系传来传去。如果一个类持有一个对象的unsafe引用的话，必须通过某种unmanaged pointer type来实现。

对于会包含unmanaged pointer types的泛型或resilient value类型，以上规则不适用。分析会假定对泛型或resilient value type执行copy_addr，会生成一个independently-managed的值。这种扩展机制可以使得使用这种类型更方便；但并是说它就确保了语言的安全性，最终责任还是在用户身上。
