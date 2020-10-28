# Functions

```
decl ::= sil-function
sil-function ::= 'sil' sil-linkage? sil-function-attribute+
                   sil-function-name ':' sil-type
                   '{' sil-basic-block+ '}'
sil-function-name ::= '@' [A-Za-z_0-9]+
```

SIL函数使用关键字sil来定义。SIL函数名以@符号开始，后面接字母数字构成的id。这个函数名也是LLVM IR中该函数的名字，但通常会使用原始的swift声明进行mangle编码。 这个sil句法声明了函数的名字，SIL的类型并且定义了花括号里的函数体。声明的类型必须是函数类型，当然也有可能涉及泛型。

## Function Attributes

```
sil-function-attribute ::= '[canonical]'
```

该函数已经是canonical SIL，即使这个模块目前还只是在Raw SIL的状态。


```
sil-function-attribute ::= '[ossa]'
```

该函数遵循 OSSA(ownership SSA)的形式。


```
sil-function-attribute ::= '[transparent]'
```
透明函数(transparent function)总是内联的，并且内联后不会保留源码信息。



```
sil-function-attribute ::= '[' sil-function-thunk ']'
sil-function-thunk ::= 'thunk'
sil-function-thunk ::= 'signature_optimized_thunk'
sil-function-thunk ::= 'reabstraction_thunk'
```
该函数是编译器生成的thunk函数。


```
sil-function-attribute ::= '[dynamically_replacable]'
```
该函数可以在运行时替换成不同的实现。优化动作不应该对该类型的函数作任何假设，即便可以获得函数体的SIL源码也不行。


```
sil-function-attribute ::= '[dynamic_replacement_for' identifier ']'
sil-function-attribute ::= '[objc_replacement_for' identifier ']'
```
以上方式可以指定用哪个函数替换那个目标函数。


```
sil-function-attribute ::= '[exact_self_class]'
```
该函数是class的designated initializer，在该函数体内部会alloc该class的静态类型。


```
sil-function-attribute ::= '[without_actually_escaping]'
```
该函数是闭包的thunk函数，其并不会escaping。


```
sil-function-attribute ::= '[' sil-function-purpose ']'
sil-function-purpose ::= 'global_init'
```
以上的属性，表达的语义如下：
- 在第一次调用之前，任何时候都可能会有副作用
- 对同一个global_init函数的调用具有相同的副作用
- 观测initializer副作用的任意操作都必须先于对该initializer的调用

当前是成立的，如果该函数是访问全局变量时候lazy生成的一个寻址器（addressor）的话。
注意，初始化函数本身并不需要此属性。该函数是private的，并且只会从寻址器内部被调用。


```
sil-function-purpose ::= 'lazy_getter'
```
该函数是 lazy属性的getter函数，且该属性后端存储的类型为Optional。
对应的getter方法包含了一个top-level的switch_enum（或 switch_enum_addr），后者会检查是否这个lazy属性已经被计算出来。

lazy属性的getter被首次调用之后，则可保证该属性已经被计算出来，且后续调用总是会执行top-level switch_enum的一些case。


```
sil-function-attribute ::= '[weak_imported]'
```
跨模块引用该函数需要使用weak链接。


```
sil-function-attribute ::= '[available' sil-version-tuple ']'
sil-version-tuple ::= [0-9]+ ('.' [0-9]+)*
```
指定该函数最低的系统可用版本。


```
sil-function-attribute ::= '[' sil-function-inlining ']'
sil-function-inlining ::= 'never'
```
该函数绝不内联。



```
sil-function-inlining ::= 'always'
```
该函数总是内联，即使使用Onone编译参数。



```
sil-function-attribute ::= '[' sil-function-optimization ']'
sil-function-inlining ::= 'Onone'
sil-function-inlining ::= 'Ospeed'
sil-function-inlining ::= 'Osize'
```
函数优化编译属性，该属性的局部优先级高于编译命令的优化参数。



```
sil-function-attribute ::= '[' sil-function-effects ']'
sil-function-effects ::= 'readonly'
sil-function-effects ::= 'readnone'
sil-function-effects ::= 'readwrite'
sil-function-effects ::= 'releasenone'
```
指定函数的内存效果。



```
sil-function-attribute ::= '[_semantics "' [A-Za-z._0-9]+ '"]'

```
函数指定的high-level语义。优化器可以在内联之前，使用这些信息进行high-level的优化。
比如，Array操作可以注解一些语义的属性，使得优化器可以进行冗余的边界检查消除优化和其他类似的优化。



```
sil-function-attribute ::= '[_specialize "' [A-Za-z._0-9]+ '"]'

```

指定哪些类型指定的代码应该被生成。


```
sil-function-attribute ::= '[clang "' identifier '"]'

```

The clang node owner.





