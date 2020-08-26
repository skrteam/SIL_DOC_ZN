# SIL Types

```
sil-type ::= '$' '*'? generic-parameter-list? type
```

SIL 类型以$符号开头，SIL的类型系统和Swift的是高度相关的，所以$后面的类型名称大体上是来自于Swift的类型语法。

## Type Lowering 

类型降级是指把FormalType降级为SILTypes。SIL Types也被成为Lowered Types。
FormalType是服务于Swift AST中的类型系统。它的设计初衷是抽象和统一句法的表示，比如对所有权转移规范和传递参数的规范用统一的规则进行表示。
另一方面，SIL的设计是为了更好地描述实现细节，从而这也要求SIL有一套不同的类型系统。

// ？？？
需要注意的是，有些情况是不需要做类型降级的。比如场景：

- 属于thin类型的
- 可能有非Swift调用的
- 可能在接口上使用bridged类型的
- 可能其Ownership行为与Swift默认的规范会有不同的


## Abstraction Difference 

泛型函数具有泛型参数，这个泛型参数是间接地注入的。
例如，为泛型参数的类型开辟足够的内存空间，并把该内存的指针传进去。

考虑如下的泛型函数：

```
func generateArray<T>(n : Int, generator : () -> T) -> [T]
```

函数参数generator返回一个类型T的结果，这个结果因为类型是未知的，所以必须通过指针间接地传递。


虽然有可以表示任意类型的值的实现方案，但可能并不是我们想要的：
- 我们不希望对于每个类型T都生成一份generateArray的具体实现（类似静态多态）
- 我们不想定义一套通用的类型表示系统去表示每一个类型
- 我们不希望因为T的不同，而导致我们必须动态地构造不同的方式去调用函数

但是，我们同样不希望泛型系统的实现最终导致灵活的代码最后变成了低效的非泛型代码。
举个例子，对于函数类型 （）->Int 我们希望能直接返回类型Int；另外，（）->Int是泛型表示 ()->T 的一个合法的类型替换表示，generateArray < Int >的调用者应该可以直接传入任意的 （）->Int类型的参数作为generator。



因此，formal type在泛型上下文的抽象表示和在类型替换之后的表示会存在差异。我们称之为 *抽象差异* 。

SIL类型系统的设计是希望通过一套SILTypes能够统一地表示这种抽象差异。对于嵌套的每一层的类型替换，都有合适的抽象的SILtypes可以进行表示。

为达到这个目标，一个泛型实体的formal type做降级的时候，必须用一个抽象化pattern, 这个pattern由尚未进行类型替换的formal types构成。
例如：

```
struct Generator<T> {
  var fn : () -> T
}
var intGen : Generator<Int>
```

例子中，intGen.fn的替换类型用formal type表示就是 () -> Int, 如果把它进行类型降级，则可以得到 @callee_owned () -> Int。 我们可以看到这个函数会直接返回一个Int类型的返回值。

但如果用未进行类型替换的泛型抽象表示我们会得到 () -> T, 对它进行降级，得到@callee_owned () -> @out Int。 我们可以看到，函数返回类型变成了@out Int，不是直接返回Int了。

如果在类型降级的时候用到了 未进行类型替换的抽象pattern进行表示的话，那就需要递归地用类型变量去替换pattern中未替换类型的泛型变量。

比如, 如果变量g的类型是Generator<(Int, Int) -> Float>，那么g.fn在类型在降级的时候，先用抽象pattern表示为，（）-> (Int, Int) -> Float. 
最终降级为SIL类型：

```
@callee_owned () -> @owned @callee_owned (@in (Int, Int)) -> @out Float.
```

再举个例子，变量h的类型是Generator<(Int, inout Int) -> Float>。h.fn的降级类型如下：

```
@callee_owned () -> @owned @callee_owned (@in Int, @inout Int) -> @out Float
```

通过多层的替换和展开，就达成了用一套SIL Types来表示具有抽象差异（Abstraction Difference）的泛型表示，并进行类型替换和类型降级了。

SILGen负责解析和生成抽象模式（Abstraction Pattern）。

目前，只有function和tuple类型存在抽象差异的问题。


```
F.Y.I.
Int 为trivial/loadable类别，可直接通过寄存器传递
而泛型T由于类型未知，是address-only的类别，只能通过指针传递
以上差异导致了调用方式和返回的方式不同

该问题通过reabstraction stub解决，见文章XXX

```

## Legal SIL Types

一个合法的SIL的值：
- $T, 表示一个对象类型，其中T是一个合法的loabable type
- $*T, 表示一个地址类型，其中T是一个合法的SIL 类型（loadable 或者 address-only类型）


一个合法的SIL类型T，包括：
- function，要求满足下列约束
- metatype，用来表达类型的特点
- tuple，要求其元素合法
- Optional< U >, 要求U合法
- @box类型，其包裹了一个合法的SIL类型
- 除以上所列，合法的Swift类型（非左值类型）

需要注意的是，在递推替换降级过程中，未被替换降级的部分，还是formal types。


## Address Types

$*T就是$T的指针。可以是一个数据结构的内部指针。可以读写loadable types类型的地址。

$T的T不能是Address-Only类型。Address-Only类型的地址只能通过相关的指令进行操组哦，比如 copy_addr,destroy_addr，也可以作为函数的参数，但是不能作为AdressType的值。

Address类型不是指向Class对象的指针类型，不能进行release和retain，不能进行引用计数。

Address类型不是语言的一等(first-class)成员。递归的写法表示类型是非法的，例如$**T是非法的。

虽然$**T不是一个合法的类型表示，但Address的地址是可以直接访问的。可以对齐进行alloc,load,store等操作。

Addresse 类型可以作为函数的indirect入参，但不可作为返回值类型。


## Box Types

被捕获的本地变量，以及非直接访问访问的值类型(indirect value types)的payloads 存储在堆内存上。

@box T包裹了一个T类型的可变值，该容器支持引用计数，并且是使用swift-native的引用计数（区别于oc运行时）。因此box类型也可以转成Builtin.NativeObject类型进行访问。


## Metatype Types

一个具体的SIL类型，必须说明它的Metatype属性：
- @thin, 该类型不需要额外的外部存储，已经可以必要地表达一个具体类型了
- @thick, 该类型存储的是一个引用，这个引用指向外部的具体类型
- @objc, 表示这个类型是OC的对象类型


## Function Types

SIL的function类型也有细分差异：

- escape 特性
  - 函数声明的参数列表中，包含有noescape类型的function类型入参的话，该参数在降级时会被标注为@convention(thin) 或者 @callee_guaranteed. 这种入参的生命周期可能超过函数本身，因此入参需要独立的上下文由外部独立管理，也就是unowned.

```
F.Y.I.
    function类型作为入参的 escape特性默认值，对于Swift：
        <= 3.0  @escape as default
        >= 4.0 @nonescape as default

        ref： https://github.com/apple/swift-evolution/blob/master/proposals/0103-make-noescape-default.md
```

- SIL function类型声明了调用的上下文细节：
  - 