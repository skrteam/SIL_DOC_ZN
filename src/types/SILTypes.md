# SIL Types

```
sil-type ::= '$' '*'? generic-parameter-list? type
```

SIL 类型以\$符号开头，SIL的类型系统和Swift的是高度相关的，所以$后面的类型名称大体上是来自于Swift的类型语法。

## Type Lowering 

类型降级是指把FormalType降级为SILTypes。SILTypes也被称为Lowered Types。
FormalType是服务于Swift AST中的类型系统。它的设计意图是抽象和统一句法的表示，比如对所有权转移规范和传参直接性等问题提供一套表示的方案。
另一方面，SIL的设计是为了更好地描述实现细节，从而这也要求SIL有一套不同的类型系统。

需要注意的是，有些情况下的声明是不需要做类型降级的。比如场景：

- 通常属于thin类型的
- 可能有非Swift调用约定的
- 可能在接口中使用bridged类型的
- 可能Ownership转换行为与Swift默认规范不同的


## Abstraction Difference 

泛型函数具有未绑定类型(uncontrained type)时，必须间接地使用它们。
例如，为未绑定类型开辟足够的内存空间，并传递该内存的地址作为指针使用。

考虑如下的泛型函数：

```
func generateArray<T>(n : Int, generator : () -> T) -> [T]
```

函数参数generator返回一个类型为T的结果，这个结果因为类型是未知的，所以必须将该结果写入一个地址，并通过一个隐式的参数进行传递。
想要处理任意类型的值，其实也并没有其他特别好的替代方案：
- 我们不希望对于每个类型T，都生成一份generateArray的具体实现
  // FYI： 类似静态多态
- 我们不想给语言里的每个类型一个通用的表示
- 我们不希望因为T的不同，而导致我们必须动态地构造不同的generator调用

但是，我们也不希望泛型系统的存在给导致非范型代码变得低效。
举个例子，对于函数类型 （）->Int 我们希望能直接返回类型Int；另外，（）->Int是泛型表示 ()->T 的一个合法的类型替换，generateArray < Int >的调用者应该可以直接传入任意的 （）->Int类型的参数作为generator。


因此，formaltype在泛型上下文中的表示和在类型替换中的表示会存在差异。我们称之为 *抽象差异（Abstraction Difference）* 。

SIL类型系统的设计希望抽象差异的具体表现能通过SIL类型表达出来。对于嵌套的场景，能用合理抽象的值来表示每一层类型替换。

为达到这个目标，一个泛型实体（generic entity）的formal type做降级的时候，总是用一个抽象模式（abstraction pattern）, 这个模式由未进行类型替换（unsubstituted）的formal types构成。
例如：

```
struct Generator<T> {
  var fn : () -> T
}
var intGen : Generator<Int>
```

例子中，intGen.fn的替换类型用formal type表示就是 () -> Int, 如果把它进行类型降级，则可以得到 @callee_owned () -> Int。 我们可以看到这个函数会直接返回一个Int类型的返回值。

但如果用未进行类型替换的泛型抽象表示我们会得到 () -> T, 对它进行降级，得到@callee_owned () -> @out Int。 我们可以看到，函数返回类型变成了@out Int，不是直接返回Int了。

在类型降级中使用非约束类型来构成抽象模式，这个模式看起来就像是一个共享的结构，只需要用不同的类型变量填入到相应的可填充类型（materializable type）中。

比如, 如果g的类型是Generator<(Int, Int) -> Float>，那么g.fn在类型在降级的时候，先用(Int, Int) -> Float 来表示T，使用抽象模式是() -> T。 这和使用抽象模式 U -> V是一样。
最终降级的SIL类型表示为：

```
@callee_owned () -> @owned @callee_owned (@in (Int, Int)) -> @out Float.
```

再举个例子，变量h的类型是Generator<(Int, inout Int) -> Float>。其中，类型(Int, inout Int)和 inout Int都不是可填充（materializable）类型，因此他们都不是类型替换的可能的结果。最后h.fn的降级类型如下：

```
@callee_owned () -> @owned @callee_owned (@in Int, @inout Int) -> @out Float
```

通过多层的替换和展开，就达成了用一套SIL Types来表示具有抽象差异（Abstraction Difference）的泛型表示，并进行类型替换和类型降级了。（This system has the property that abstraction patterns are preserved through repeated substitutions. That is, you can consider a lowered type to encode an abstraction pattern; lowering T by R is equivalent to lowering T by (S lowered by R).）

SILGen负责解析和生成抽象模式（Abstraction Pattern）。

目前，只有function和tuple类型存在抽象差异的问题。


```
FYI
Int 为trivial/loadable类别，可直接通过寄存器传递
而泛型T由于类型未知，是address-only的类别，只能通过指针传递
以上差异导致了调用方式和返回的方式不同

该问题通过reabstraction stub解决，见文章XXX

```

## Legal SIL Types

一个合法的SIL的值：
- $T, 表示一个对象类型，其中T是一个合法的loadable type
- $*T, 表示一个地址类型，其中T是一个合法的SIL类型（loadable 或者 address-only类型）


一个合法的SIL类型T，包括：
- function，要求满足下列约束
- metatype，用来描述类型的表示（representation）
- tuple，要求其元素合法
- Optional< U >, 要求U为合法的SIL类型
- @box类型，其包裹了一个合法的SIL类型
- 除以上所列，合法的Swift类型（非左值类型）

需要注意的是，在别的递归位置的类型在类型语法上还是formal types。
例如，一个metatype的实例类型或者一个范型的类型参数，它们仍然是formal types，而不是降级的SILTypes.


## Address Types

$*T就是$T的指针。可以是一个数据结构的内部指针。可以读写loadable types类型的地址。

$T的T不能是Address-Only类型。Address-Only类型的地址只能通过相关的指令进行操作，比如 copy_addr,destroy_addr，也可以作为函数的参数，但是不能作为AddressType的值。

Address类型不支持引用计数，不像Class的值可以release和retain。

Address类型不是语言的一等(first-class)成员。递归的写法在类型表达式中是非法的，例如$**T是非法的。

地址的地址不能直接地获取。$**T不是一个合法的类型表示。Address类型的值因此不能被allocated，loaded，或者stored（虽然地址是可以的）。

Addresse 类型可以作为函数的indirect入参，但不可作为返回值类型。


## Box Types

被捕获的本地变量，以及存储在堆内存上的间接访问值类型(indirect value types)的payloads。

@box T 是一个容器，包裹了一个T类型的可变值。该容器支持引用计数，使用swift-native的引用计数（区别于oc运行时）。因此box类型也可以转成Builtin.NativeObject类型进行访问。


## Metatype Types

一个具体的SIL类型，必须说明它的Metatype属性：
- @thin, 该类型不需要额外的外部存储，就可以必要地表达一个具体类型（仅可用于具体的metatypes）
- @thick, 该类型存储个引用，该引用指向一个类型或者该类型的子类（如果是一个具体的class类型的话）
- @objc, 该类型存储个引用，该引用指向一个OC的对象类型（或者它的子类），而不是Swift对象的表示。


## Function Types

SIL的function类型和Swift的function类型有多处不同：
- SIL function类型可能是泛型函数。例如，使用function_ref指令访问泛型函数会得到一个泛型函数类型的值。
- SIL function类型可能声明为noescape
  - 函数声明的参数列表中，包含有noescape类型的function类型入参的话，该参数在降级时会被标注为@convention(thin) 或者 @callee_guaranteed. 这种情况属于unowned context,即context的生命周期必须保证能独立处理好.

```
FYI
    function类型作为入参的 escape特性默认值，对于Swift：
        <= 3.0  @escape as default
        >= 4.0 @nonescape as default

        ref： https://github.com/apple/swift-evolution/blob/master/proposals/0103-make-noescape-default.md
```

- SIL function类型可以声明对于context值使用何种调用约定来进行处理：
  - 对于@convention(thin)，表示函数没有context的值。这种类型也可声明为@noescape,即如果有context值，传递了也不会有额外的效果。
  - 对于@callee_guaranteed,表示context的值作为直传(direct)参数处理，且使用@convention(thick)的方式处理。若同时函数类型声明为@noescape，那么context的值是unowned;反之（@escaping），则context的值为guaranteed。
  - 对于@callee_owned，表示context的值为一个owned直传参数。且使用@convention(thick)方式处理。注意它并且与@noescape互斥，不可同时使用。
  - 对于@convention(block),表示context的值为unowned直传参数。
  - 其他的函数类型的规范会在Properties of Types和 Calling Convention部分讨论。


- SIL function类型声明了参数的调用约定。参数列表使用无标签元组(unlabeled tuple)的数据结构；tuple的每个元素必须是合法的SIL类型，并可以可选地添加以下某一种属性进行修饰：
  - 直传参数的值类型用T表示，间接参数的值类型用*T表示。
  - @in修饰间接参数。地址必须指向一个已初始化的对象；函数负责销毁所指对象。
  - @inout修饰间接参数。地址必须指向一个已初始化的对象。需保证在函数调用返回前该地址保持被初始化的状态。函数可以修改指针所指（pointee），且可以弱假设不会发生对该实参进行别名化的读写（aliasing reads）。这要求该实参的值保持有效的状态，以便有序的别名违例(aliasing violations)不会影响内存安全。由此可以进行优化，如：local load/store propagation；引入和消除临时拷贝；把@inout参数提升为一个@owned直传参数和一个返回结果的方案。但是不允许优化导致该内存处于非初始化的状态。
  
```
FYI
  Pointer Aliasing
  多个指针的类型和所指向的地址是相同的，难以区分的情况，可能会引起问题。
  https://en.wikipedia.org/wiki/Pointer_aliasing
```

  - @inout_aliasable修饰间接参数。地址指向已初始化的对象。需保证在函数返回前该地址始终是已初始化的。函数可以修改指针所指，同时必须假定参数的别名也会修改指针所指。可以假设这些别名的类型是正确的，且访问操作也是有序的。而不正确的类型（ill-typed）访问和数据竞争的结果是未定义的。
  - @owned修饰owned直传参数。
  - @guaranteed修饰guaranteed的直传参数。
  - @in_guaranteed修饰间接参数。地址指向已初始化的对象。确保调用者和被调者都不修改指针所指，但被调用者（callee）可以读取。
  - @in_constant修饰间接参数。地址指向已初始化的对象。函数中该值read-only。
  - 此外，都是unowned直接参数。


- SIL function类型声明了调用结果（result）的约定规范。结果列表会写成unlabeled tuple；tuple的每个元素必须是合法的SIL类型，并可以可选地用下列属性进行修饰。有直接和间接结果两种（Indirect and direct results may be interleaved）.

```
FYI 
原文中的result： 包括了 返回值 和 inout类型的参数
```

间接结果会出现在函数的entry block的隐式参数中，以及在apply和try_apply指令的参数中，用*T来表示。这些参数的顺序和在结果列表中的顺序一致，并且总是位于其他参数之前。

直接结果的类型表示为T。如果只有一个直接结果，那么返回结果类型就是这个值的类型；此外，返回类型是一个tuple类型的结果列表，tuple包含了所有直接返回结果。返回类型同时也是return指令、apply指令的操作数类型，以及try_apply指令的普通结果（normal result）的类型。

  - @out，修饰间接结果。地址必须是未初始化的对象；函数负责初始化内存为一个有效的值。除非发生异常，使用throw指令退出，或者函数使用非Swift的调用约定。
  - @owned，修饰owned的直接结果。
  - @autoreleased，autoreleased的直接结果。必须是唯一的直接结果。
  - 其他，unowned直接结果。

trivial类型的直传参数或结果总是unowned。

owned直接参数或者结果会把ownership转移给接受者，由其负责销毁。这意味着在值进行传递的时候，会引用计数会+1。

unowned的直接参数和结果，在传递的当下是有效的。接受者不用担心竞争条件会立即销毁该值，但还是应该先copy(通过strong_retain指令)再使用.

guaranteed的直接参数和unowned直接参数类似，不同在于调用者需要保证该值在调用过程中始终有效。由此可知，任何在被调函数中出现的strong_retain, strong_release指令对都可以优化消除。

autoreleased返回值的类型必须是支持retain的指针表示。autoreleased的结果在传递的时候不会在名义上对引用计数+1，但在运行时会安全地进行+1处理。这些处理要求精确的代码布局控制（Code-layout Control）。在SIL Pattern的表示上，autorelease的表示和owned表示看起来差不多，并且在SIL降级到LLVM IR的阶段，会在这两层的代码中插入额外的运行时指令。
对具有autoreleased结果的函数进行autoreleased调用(apply)，其结果在传递过程中引用计数会+1。
对定义不具有autoreleased结果的函数进行autoreleased调用(apply)，其结果引会被调用者进行强引用。
对具有autoreleased结果的函数进行non-autoreleased调用(apply)，其结果被被调者（callee）进行autorelease处理。

- SIL 函数类型可以接受一个可选的Error结果，写作@Error。Error结果总是具有隐式的@Owned属性。仅适用于native call规范。
  - 具有Error结果的函数必须使用try_apply指令进行调用。除非编译器可以证明函数实际不会throw，那么可以使用apply [nothrow]。
  - 用return指令返回普通结果。用throw返回error结果。
  - 对formal function类型降级的时候，throws注解会转化成更加具体的Error传播形式
    - 对于native Swift函数，throws会转换成一个error结果
    - 其他情况，会基于导入的API，将throws转换成显式的错误处理机制。导入器只会导入throws注解的非原生的方法和类型。如果可能的话，会自动这样处理。
    

- SIL函数类型提供了一种模式化签名(pattern signature)和模式替换（pattern substitution），通过特定的泛型抽象模式来表达该类型的值。这两者需要同时提供。如果使用了模式化签名，那么对应部分的类型（参数、结果、yields）都必须用泛型参数的签名方法来表示。另一方面，模式替换则应该用总体泛型签名的泛型参数来表达，或者使用闭包化的泛型上下文。
  - 模式签名在@substituted属性之后，它必须是函数类型前的最后的一个属性。
  - 模式替换跟在函数类型之后，在关键词 for 之后。
  - 如下例：

```
@substituted <T: Collection> (@in T) -> @out T.Element for Array<Int>
```

该类型的值的低级表示可能和它在替换过程中的版本不匹配。如下例：
```
(@in Array<Int>) -> @out Int
```

convert_function指令可以对函数值在最外层的模式替换中的差异进行调整。注意，这只作用于最外层，而不会对嵌套的情况进行处理。比如，对于上面例子中两个类型，如果一个函数将前者作为参数类型，是不能直接转成后者的类型的。要完成这样的转换，要借助于thunk函数。

对函数类型的模式签名进行类型替换只会对其类型占位符进行替换；构件（Component）类型会保留原有的结构。

- 在实现上，SIL函数类型可以携带泛型签名的替换信息。这可以给泛型类型的应用和实现带来一些方便，但不算做SIL语言的正式标准。照理说泛型的值不应该携带这些类型。这样的类型仿佛是替换了底层的函数类型，其表现就像是非泛型类型。

## Async Functions

SIL函数包括了@async类型。 @sysnc函数会运行内部的异步任务，并且可以显式地指定在一些代码点挂起程序执行。@async函数只能被其他@async函数调起，或者通过apply和try_apply指令来调用（或者如果是协程的话，可以通过begin_apply指令调起）

在Swift中，withUnsafeContinuation原语（primitive）用来实现执行的挂起点。在SIL中，@async函数使用get_async_continuation\[_addr\]指令和await_async_continuation指令来表示该抽象。get_async_continuation\[_addr\]可以访问一个continuation value，在协程被挂起之后，可以使用后者来恢复执行。该resulting continuation value会被传入一个completion handler,并注册到一个event loop里，或者有其他别的机制对其进行调度。
Continuation上的操作可以在async 函数恢复执行的时候，将一个值或者一个error传回到async函数中。 await_async_continuation指令会挂起协程的执行流，等待continuation将其唤醒恢复执行。


如下是withUnsafeContinuation在Swift中的使用：

```
func waitForCallback() async -> Int {
  return await withUnsafeContinuation { cc in // FYI cc为 continuation context
    registerCallback { cc.resume($0) }
  }
}
```

可能会降级为如下的SIL：

```
sil @waitForCallback : $@convention(thin) @async () -> Int {
entry:
  %cc = get_async_continuation $Int
  %closure = function_ref @waitForCallback_closure
    : $@convention(thin) (UnsafeContinuation<Int>) -> ()
  apply %closure(%cc)
  await_async_continuation %cc, resume resume_cc

resume_cc(%result : $Int):
  return %result
}
```

必须保证每个continuation value只对它对应的async coroutine使用一次。如果超过一次，则程序行为是未知的。另一方面看，如果没有成功恢复其执行，那么异步任务会卡在挂起的状态，相应的所属资源和内存都会泄漏。


## Coroutine Types

协程(Coroutine)是指一个函数可以把它自己挂起，并把执行流的控制权交给调用者，而不用终止当次函数调用。因此，它可以不遵循严格的栈规则。SIL协程的控制流和其调用者紧密结合在一起。在yield点，可以以一种结构化的方式将值在caller和callee间进行双向传递。
Swift中的Generalized accessor和generator需要符合如下描述：
一个 read 和 modify的 accessor coroutine会映射到一个值，并且可以通过yield暂时地把该值的ownership交出来给caller。当coroutine恢复时，再把值的ownership收回。这样就使得coroutine可以清理相关资源，并对外部修改该值的行为作出相应的处理。
Generator的情况类似，它是流式yield值，但一次只yield一个值，同样临时交出值的ownership。 调用者控制流和这些coroutine的紧密耦合使得外部可以从coroutine借用（borrow）值。对于通常的普通函数而言，函数返回会转移返回值的ownership，因为在函数返回后其上下文就已经不存在了，也就不能继续维护返回值的ownership。

```
FYI  理论上，Coroutine可以把控制权交给任何人。
```

为了支持以上的这些概念，SIL支持两种协程：@yield_many 和 @yield_once。 将该属性写在函数类型的前面，以标志这个函数是一个协程类型。
@yield_many 和 @yield_once coroutine和@async关键词是兼容的。(您可以注意到但在SIL中@async函数本省不会显式地作为一种coroutine模型，只是在async在降级的时候使用coroutine作为它的实现方式。)


协程类型可以声明任意数量的yield的值。也就是说，yield给出的值都要声明在函数原型中。具体是写在函数类型的结果列表中，@yields作为前缀关键词。一个被yield的值会包含一个调用属性，该属性包含了一些调用参数，看起来就像从yield处调出，处理完又掉回该处。


目前，协程不会产生普通的结果（normal result）.


协程函数可以想普通的函数一样有很多种使用方式。但不能使用标准的apply 或者 try_apply指令来调用。对于non-throwing 且 yield-once的协程，使用begin_apply指令调用。目前对于 throwing的协程 或者 yield-many的协程，暂时不支持调用。


协程可能包括特殊的 yield 和 unwind 指令。


@yield_many协程可以yield调出任意多次。@yield_once在函数返回前只能yield一次，但它可以在返回前throw错误。


## Properties of Types

根据ABI稳定性和泛型约束，把SIL类型分类成了以下几组：

- loadable类型, 指可以完全暴露地用SIL进行具体表示的类型
  - Reference types
  - Builtin value types
  - Fragile struct types in which all element types are loadable
  - Tuple types in which all element types are loadable
  - Class protocol types
  - Archetypes constrained by a class protocol

loadable类型的值可以对其本身或独立的子结构进行load和store。因此：
  - loadable类型的值可以load进SIL SSA的值中，并且store操作不需要运行额外的用户编码，需注意编译器会自动生产ARC代码。
  - 对loadable类型的值进行比特序拷贝，就可以获得一个初始化好的拷贝值


loadable的聚合类型包括了loadable的struct和tuple。


trivial类型是具有trivial 值语义的loadable类型。而且trivial 的load和store操作不需要ARC，其值也不需要销毁。



- Runtime-sized types,是指在编译器无法获得静态尺寸的类型，包括：
  - Resilient value types 
  - Fragile struct or tuple types that contain resilient types as elements at any depth
  - Archetypes not constrained by a class protocol

- Address-only types,是指不能load或以其他方式作为SSA值使用的类型
  - Runtime-sized types
  - Non-class protocol types
  - @weak types
  - 不满足loadable的要求的类型。比如某些值和内存位置相关联，在copy和move的时候需要运行一些用户定义的编码。最常见的情况是，这些值注册在全局作用域的数据结构上，或者其包含了指向自身的指针。比如：
    - @weak标记的弱引用值的地址被注册在一个全局表中。当该@weak值被copy和move到新的地址时，全局注册表也需要更新。
    - 堆上不支持COW(CopyOnWrite)的集合类型对象，（就像C++的std::vector），当集合被copy的时候，集合元素也需要一起被copy到新的内存地址。
    - 不支持COW的String类型，但通过持有指向自己的指针来实现一些优化的情况（就像C++里的std::string）。当String值被copy和move之后，该内部指针也需要更新。

address-only类型的值必须保证始终在内存里，SIL要使用只能通过内存地址引用。该类型的地址不可以进行load和store操作。SIL提供了一些专门的指令，间接地操作address-only类型，比如copy_addr和 destroy_addr。


另外，还有一些可以被分类的类型：
- 堆对象引用类型：其表示包含一个强引用计数的指针。包括：所有的class类型，Builtin.NativeObject，AnyObject，遵循class protocol的archetype。
- 引用类型在低级表示中可以包含额外的全局指针，和一个强引用计数指针。该类型包括所有的堆对象引用类型，thick函数类型，以及遵循class protocol的protocol的构件（composition）类型。所有引用类型都可以被retain和release，也可以拥有ownership语义的表示。
- 具有retainable指针表示的类型可以兼容ObjC的id类型。在运行时该值可能为nil。这包括了classes, class metatypes, block functions， class-bounded existentials with only Objective-C-compatible protocol constraints，还有用Optional或者ImplicitlyUnwrappedOptional对以上类型进行包装的类型。具有retainable的指针表示的类型可以使用@autoreleased的规范进行return。


SILGen不能保证将Swift函数类型一一映射到SIL函数类型。函数类型的转换就是给其编码一些额外的属性：

- 关于调用约定的语义，通过以下属性表示：

```
@convention(convention)
```
该属性和语言层面的@convention有点像，但在SIL层扩展了多个额外的版本：
  - @convention(thin)，thin函数：使用swift调用约定，并且不会额外传递"self"或"context"参数
  - @convention(thick)，thick函数：使用swift调用约定，并且传递一个引用计数的context对象。context捕获上下文信息或者持有函数的状态。该属性具有 @callee_owned 或者 @callee_guaranteed的Ownership语义
  - @convention(block)，ObjC兼容的block。兼容ObjC的ID类型，其对象包含了block的调用函数，其使用C的调用约定。
  - @convention(c)，C函数：不携带context，使用C函数调用约定：使用C调用约定，在SIL层会携带一个self参数，在实现层面，映射为self和_cmd参数。
  - @convention(objc_method)，使用ObjC方法实现。
  - @convention(method)，Swift实例方法实现。使用Swift调用约定，携带特殊的self参数。
  - @convention(witness_method)，Swift协议方法实现。通过这样一种函数的多态约定方式，来确保该协议的所有实现者都可以通过多态的方式使用该函数。

