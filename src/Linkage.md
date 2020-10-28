# Linkage

```
sil-linkage ::= 'public'
sil-linkage ::= 'hidden'
sil-linkage ::= 'shared'
sil-linkage ::= 'private'
sil-linkage ::= 'public_external'
sil-linkage ::= 'hidden_external'
sil-linkage ::= 'non_abi'
```
链接描述符控制了不同SIL模块中的对象如何链接，也就是如何当作同一个对象。


一个链接如果有 external的后缀，那么它是external的。 一个对象的链接如果不是external的话，那么它在当前模块内必须有定义。


所有的函数，全局变量，以及vitness table 都有链接。 定义的默认链接描述定义为public；而声明的默认值为public_external。（有可能以上默认值逐渐会对应改为hidden和hidden_external。）

对于全局变量的外部链接意味着该变量不是一个定义。一个变量如果缺少显式的链接描述符则会被推断为一个定义（因此会获得定义的默认链接描述，public）。


## Definition of the linked relation

如果对象如果互相可见，并且具有相同的名字，那么就会被链接起来。

- 具有public 或 public_external链接描述的对象总是可见的
- 具有hidden 或 hidden_external，或shared 链接描述的对象只对模块内的对象可见
- 具有private 链接描述的对象只对模块内的对象可见


请注意，链接关系是等价关系：它是自反的，对称的和可传递的。

## Requirements on linked objects

要使两个对象链接，他们必须具有相同的类型。

要使两个对象链接，他们必须具有相同的链接描述，除了：
- public对象可以链接到public_external对象上
- hidden对象可以链接到hidden_external对象上

要使两个对象链接，则两者至多有一个是定义，除非：
- 两者都具有shared属性
- 至少其一具有external属性


如果两个对象都是定义，且链接在一起了，那么这两个定义在语义上必须是等价的。这种等价可能仅存在于定义良好的代码的用户可见语义上；但不应该认为其保证链接定义在操作上也完全等效。
例如，一个函数的定义可能会从地址参数中复制一个值，而另一个函数分析证明了并不需要该值。


如果一个对象被使用了，那么它必须被链接到一个with non-external 的定义上。


## Public non-ABI linkage

non-abi链接是比较特别的链接，它仅提供给序列化的SIL中的定义使用的，它不会在object文件中定义可见的符号。

具有non-abi链接描述的定义有点像shared链接，区别在于它必须被序列化，即使没有被当前模块的任何位置所引用。例如，它可能会被当作无用函数消除优化（dead function elimination）的根。

当一个non-abi的定义被反序列化后，它将具有shared_external链接描述。

并没有non_abi_external这种链接描述。相反，当需要引用同一Swift模块但不同转换单元中定义的non_abi声明时，必须使用hidden_​​external链接描述。


## Summary


- public定义在程序中是唯一的，且到处都可见。在LLVM IR中，他们被输出为external链接性以及default的可见性。
- hidden定义在当前Swift模块中唯一且可见。在LLVM IR中，他们被输出为external链接性以及hidden的可见性。
- private定义在当前Swift模块中唯一且可见。在LLVM IR中，他们被输出为private链接性。
- hidden定义在当前Swift模块中唯一且可见。在LLVM IR中，他们被输出为external链接性以及hidden的可见性。
- shared定义在当前Swift模块中可见。仅可与其他等价的shared定义链接；因此，只有确实被用到了才需要被输出（emit）。在LLVM IR中，他们被输出为linkonce_odr链接性以及hidden的可见性。
- public_external和hidden_external对对象始终在其他位置具有可见的定义。如果此对象仍有定义，那仅是出于优化或分析的目的。在LLVM IR中，其声明具有external链接性，并且定义（实际会输出为定义）具有available_externally链接性。