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

$T的T不能是Address-Only类型。Address-Only类型的地址只能通过相关的指令进行操作，比如 copy_addr,destroy_addr，也可以作为函数的参数，但是不能作为AdressType的值。

Address类型不是指向Class对象的指针类型，不能进行release和retain，不能进行引用计数。

Address类型不是语言的一等(first-class)成员。递归的写法表示类型是非法的，例如$**T是非法的。

虽然$**T不是一个合法的类型表示，但Address的地址是可以直接访问的。可以对其进行alloc,load,store等操作。

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
- SIL function类型可能是泛型函数。例如，使用function_ref指令访问泛型函数会得到一个泛型函数类型的值。
- SIL function类型可能声明为noescae
  - 函数声明的参数列表中，包含有noescape类型的function类型入参的话，该参数在降级时会被标注为@convention(thin) 或者 @callee_guaranteed. 这种入参的生命周期可能超过函数本身，因此入参需要独立的上下文由外部独立管理，也就是unowned.

```
F.Y.I.
    function类型作为入参的 escape特性默认值，对于Swift：
        <= 3.0  @escape as default
        >= 4.0 @nonescape as default

        ref： https://github.com/apple/swift-evolution/blob/master/proposals/0103-make-noescape-default.md
```

- SIL function类型可以声明如何处理context值：
  - @convention(thin)，表示函数不需要一个context值。同时也可以声明为@noescape,即使传递了context值，也不会有额外的效果。
  - @callee_guaranteed,表示context值作为一个直接参数传入，也意味着是@convention(thick)。若声明函数类型为@noescape，那么context是unowned;反之（@escaping），则context为guaranteed。
  - @callee_owned，表示context为一个owned直接参数。这意味着调用模式为@convention(thick)，并且与@noescape互斥。
  - @convention(block),表示context为unowned直接参数。
  - 其他的函数类型的规范会在Properties of Types和 Calling Convention部分讨论。
- SIL function类型可以声明如何处理调用参数。参数列表会写成unlabeled tuple；tuple的每个元素必须是合法的SIL类型，并可以可选地用下列属性进行修饰：
  - 直接参数的类型表示为T，间接参数的类型表示为*T。
  - @in修饰间接参数。地址指向已初始化的对象；函数负责销毁。
  - @inout修饰间接参数。地址指向已初始化的对象。需保证在函数返回前该地址始终是已初始化的。函数可以修改指针所指，甚至可以弱假设不会发生对该实参进行别名化的读写。虽然这要求保留该实参的一个有效值，使得有序别名违例不会和内存安全妥协。基于该辨识的分析，可以做一些优化，如：local load/store propagation；引入和消除临时拷贝；即那个@inout参数提升为@owned直接参数以及返回值。但是不允许优化导致该内存处于非初始化的状态。
  - @inout_aliasable修饰间接参数。地址指向已初始化的对象。需保证在函数返回前该地址始终是已初始化的。函数可以修改指针所指，同时必须假定参数的别名也会修改指针所指。这些别名具有完善的类型和顺序。对参数进行不符合类型规范（ill-typed）的访问和数据竞争的结果是未定义的。
  - @owned修饰owned直接参数。
  - @guaranteed修饰guaranteed的直接参数。
  - @in_guaranteed修饰间接参数。地址指向已初始化的对象。确保调用者和被调者都不修改指针所指，但被调用者可以读取。
  - @in_constant修饰间接参数。地址指向已初始化的对象。函数中该值read-only。
  - 此外，都是unowned直接参数。


- SIL function类型声明了调用结果的约定规范。结果列表会写成unlabeled tuple；tuple的每个元素必须是合法的SIL类型，并可以可选地用下列属性进行修饰。有直接和间接结果两种（Indirect and direct results may be interleaved）.

/// FYI 原文使用 result： 包括了 返回值 和 inout类型的参数

间接结果暗示了函数入口代码Block的参数类型为 *T，配合使用apply和try_apply指令。这些间接参数总是在参数列表的前部，和在结果列表中的顺序保持一致。

直接结果的类型表示为T。如果只有一个直接结果，那么返回结果类型就是这个值的类型；此外，返回类型是一个tuple类型的结果列表，tuple包含了所有直接返回结果。返回类型同时也是return指令的操作数类型、apply指令的类型，和try_apply指令的普通返回（normal result）的类型。

  - @out，间接结果。起始，地址指向未初始化的对象。地址必须是未初始化的对象；函数负责初始化内存为一个有效的值。除非发生异常，使用throw指令退出，或者函数不使用Swift的调用规范。
  - @owned，owned直接结果。
  - @autoreleased，autoreleased的直接结果。必须是唯一的直接结果。
  - 其他，unowned直接结果。

trivial类型的直接参数总是unowned。

owned直接参数或者结果会把ownership转移给接受者，由其负责销毁。这以为着在值进行传递的时候，会自动retain +1。

unowned的直接参数和结果，在传递的当下是有效的。接受者不用担心竞争条件会立即销毁该值，但还是应该先copy(通过strong_retain指令)再使用.

guaranteed的直接参数和unowned直接参数类似，不同在于调用者需要保证该值在调用过程中始终有效。由此可知，任何在被调函数中出现的strong_retain, strong_release指令对都可以优化消除。

autoreleased返回值的类型必须是支持retain的指针表示。autoreleased的结果在传递的时候不会在名义上对引用计数+1，但在运行时会安全地进行+1处理。这些处理要求精准的代码布局控制（Code-layout Control）。在SIL Pattern的表示上，autorelease的表示和owned表示看起来差不多，区别在于在做类型降级的时候，会在SIL层和LLVM IR层的指令中插入额外需要的指令。
对具有autoreleased结果的函数进行autoreleased调用(apply)，其结果在传递过程中引用计数会+1。
对不具有autoreleased结果的函数进行autoreleased调用(apply)，其结果引用计数会被调用者+1。
对具有autoreleased结果的函数进行non-autoreleased调用(apply)，其结果引用计数会在被调过程中+1。

- SIL 函数类型可以接受一个可选的Error结果，写作@Error。Error结果总是@Owned的。仅适用于native call规范。
  - 具有Error结果的函数必须使用try_apply指令进行调用。除非编译器可以证明函数实际不会throw，那么可以使用apply [nothrow]。
  - 用return指令是返回普通结果。用throw返回error结果。
  - 类型降级的时候，throws注解会转化成更加具体的Error传播形式
    - 对于native Swift函数，throws会转换成一个error结果
    - 其他情况，会根据具体的API，将throws转换成显式的错误处理机制。可能的话，Importer会将throws自动替换成非native的方法和类型。

- SIL函数类型提供了一种模式化签名(pattern signature)和类型替换方法来表达泛型类型的抽象化模式。但这两者需要一起使用。如果使用了模式化签名，那么对应部分的类型（参数、结果、yields）都必须用泛型参数的签名方法来表示。另一方面，模式替换则应该用泛型参数的泛型签名来表示，包括具有泛型的闭包上下文。
  - @substituted属性放在模式签名的最前。关键词 for 之后跟着模式的类型替换。如下例：

```
@substituted <T: Collection> (@in T) -> @out T.Element for Array<Int>
```

值类型的低级表示可能和类型替换过程中的版本不一致。如下例：
```
(@in Array<Int>) -> @out Int
```

convert_function指令可以对函数值在最外层的类型替换中的差异进行调整。注意，这只作用于最外层，而不会对嵌套的情况进行处理。比如，对于上面例子中两个类型，如果一个函数将前者作为参数类型，是不能直接转成后者的类型的。要完成这样的转换，要借助于thunk函数。

对函数类型的模式签名进行类型替换只会对其类型占位符进行替换；构件（Component）类型会保留原有的结构。

- 在实现上，SIL函数类型可以携带泛型签名的类型替换信息。这可以给泛型类型的应用和实现带来一些方便，但不算做SIL语言的正式标准。照理说泛型的值不应该携带这些类型。这样的类型仿佛是替换了底层的函数类型，其表现就像是非泛型类型。



## Coroutine Types

协程(Coroutine)是指一个函数可以把它自己挂起，并把执行流的控制权交给调用者，而不用终止当次函数调用。因此，它可以不遵循严格的栈规则。

/// FYI  广义的Coroutine可以把控制权交给任何人。这里执行流交还给调用者是类似Generator的模式，特指Swift中的定义。

SIL支持两种协程：@yield_many 和 @yield_once。 将该属性写在函数类型的前面，以标志这个函数是一个协程类型。

协程类型可以声明任意数量的yield的值。也就是说，yield的值都要声明在函数原型中。具体是写在函数类型的结果列表中，@yields作为前缀关键词。一个被yield的值会包含一个调用属性，该属性包含了一些调用参数，看起来就像从yield处调出，处理完又掉回该处。

目前，协程不会产生普通的结果（normal result）.

协程函数可以想普通的函数一样有很多种使用方式。但不能使用标准的apply 或者 try_apply指令来调用。对于non-throwing 且 yield-once的协程，使用begin_apply指令调用。目前对于 throwing的协程 或者 yield-many的协程，暂时不支持调用。

协程可能包括特殊的 yield 和 unwind 指令。

@yield_many协程可以yield调出任意多次。@yield_once在函数返回前只能yield一次，但它可以在返回前throw错误。

协程表示是对控制流控制权转移以及相关信息传递的一种封装，和调用者紧密地结合。这种机制可以满足Swift对generalized accessor和generator等特性的支持。但它不是专门针对async
-await这种异步风格的，针对这种风格，可以使用更简单的表示方式。

/// ？？？  lattner的proposal是 async-await风格  https://gist.github.com/lattner/429b9070918248274f25b714dcfc7619




