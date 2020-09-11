# 0x7 Calling Convention

## Swift Calling Convention @convention(swift)

Swift调用约定（Calling Convention）是Swift原生函数使用的。

函数输入类型里的tuple会被递归地解构成分离的实参。该过程发生在basicblock的入口处，或者在调用者使用apply指令的时候:

```
func foo(_ x:Int, y:Int)

sil @foo : $(x:Int, y:Int) -> () {
entry(%x : $Int, %y : $Int):
  ...
}

func bar(_ x:Int, y:(Int, Int))

sil @bar : $(x:Int, y:(Int, Int)) -> () {
entry(%x : $Int, %y0 : $Int, %y1 : $Int):
  ...
}

func call_foo_and_bar() {
  foo(1, 2)
  bar(4, (5, 6))
}

sil @call_foo_and_bar : $() -> () {
entry:
  ...
  %foo = function_ref @foo : $(x:Int, y:Int) -> ()
  %foo_result = apply %foo(%1, %2) : $(x:Int, y:Int) -> ()
  ...
  %bar = function_ref @bar : $(x:Int, y:(Int, Int)) -> ()
  %bar_result = apply %bar(%4, %5, %6) : $(x:Int, y:(Int, Int)) -> ()
}
```

如果函数调用的输入输出类型都是trivial类型的话，那只要简单地对参数传值就可以了。如下函数：

```
func foo(_ x:Int, y:Float) -> UnicodeScalar

foo(x, y)
```

在SIL中调用：

```
%foo = constant_ref $(Int, Float) -> UnicodeScalar, @foo
%z = apply %foo(%x, %y) : $(Int, Float) -> UnicodeScalar
```


## Reference Counts

注意⚠️本节的内容仅针对经验法则。实参的实际行为是由实参的约定属性（例如，@ owned）定义的，而不是调用约定本身。



引用类型的实参在传递的时候会被retain +1，由callee进行consume。 引用类型的返回值在return的时候会+1，由caller进行consume。
具有一些引用类型所构成的值类型，会和其成员引用类型一起retain和release。

/// FYI consumed parameters
/// consume就是负责进行reference count
/// https://clang.llvm.org/docs/AutomaticReferenceCounting.html#consumed-parameters


```
class A {}

func bar(_ x:A) -> (Int, A) { ... }

bar(x)
```

SIL调用：
```
%bar = function_ref @bar : $(A) -> (Int, A)
strong_retain %x : $A
%z = apply %bar(%x) : $(A) -> (Int, A)
// ... use %z ...
%z_1 = tuple_extract %z : $(Int, A), 1
strong_release %z_1
```

当callee是一个thunk函数时，函数值也会被thunk consume，并且retain +1。


## Address-Only Types

对于Address-Only类型的实参，caller会alloc生产一个拷贝副本并且把副本的地址传递给callee。callee接管副本的ownership，并负责对其销毁或consume，而caller仍然负责dealloc对应的内存。
对于Address-Only的返回值，caller alloc一块未初始化的内存作为buffer，并且把该地址作为第一个参数传递个callee。而callee必须在返回前初始化这块buffer。如下：

```
 @API struct A {}

func bas(_ x:A, y:Int) -> A { return x }

var z = bas(x, y)
// ... use z ...
```

在SIL中是这样的：

```
%bas = function_ref @bas : $(A, Int) -> A
%z = alloc_stack $A
%x_arg = alloc_stack $A
copy_addr %x to [initialize] %x_arg : $*A
apply %bas(%z, %x_arg, %y) : $(A, Int) -> A
dealloc_stack %x_arg : $*A // callee consumes %x.arg, caller deallocs
// ... use %z ...
destroy_addr %z : $*A
dealloc_stack stack %z : $*A
```

@bas的实现负责consume %x_arg，并且初始化%z。

Tuple类型是实参会被拆解，而忽略tuple类型是不是address-only的。其拆解出来的多个field会根据上述调用约定被独立地传递。如下：

```
@API struct A {}

func zim(_ x:Int, y:A, (z:Int, w:(A, Int)))

zim(x, y, (z, w))
```

在SIL里看起来是这样的：

```
%zim = function_ref @zim : $(x:Int, y:A, (z:Int, w:(A, Int))) -> ()
%y_arg = alloc_stack $A
copy_addr %y to [initialize] %y_arg : $*A
%w_0_addr = element_addr %w : $*(A, Int), 0
%w_0_arg = alloc_stack $A
copy_addr %w_0_addr to [initialize] %w_0_arg : $*A
%w_1_addr = element_addr %w : $*(A, Int), 1
%w_1 = load %w_1_addr : $*Int
apply %zim(%x, %y_arg, %z, %w_0_arg, %w_1) : $(x:Int, y:A, (z:Int, w:(A, Int))) -> ()
dealloc_stack %w_0_arg
dealloc_stack %y_arg
```

## Variadic Arguments

不定长参数以及tuple的元素会被打包成一个数组，并且作为一个单独的实参进行传递。如下：

```
func zang(_ x:Int, (y:Int, z:Int...), v:Int, w:Int...)

zang(x, (y, z0, z1), v, w0, w1, w2)
```

其SIL如下：

```
%zang = function_ref @zang : $(x:Int, (y:Int, z:Int...), v:Int, w:Int...) -> ()
%zs = <<make array from %z1, %z2>>
%ws = <<make array from %w0, %w1, %w2>>
apply %zang(%x, %y, %zs, %v, %ws)  : $(x:Int, (y:Int, z:Int...), v:Int, w:Int...) -> ()
```


## @inout Arguments

@inout实参通过地址传递 在到函数的入口处。callee不会接管引用内存的ownership。在函数入和退出时，其引用内存必须是已初始化的状态。
如果@inout实参引用里一个fragile physical变量，那么这个实参是这个变量的地址。
如果@inout实参引用的是一个逻辑属性，那么这个实参是caller-owned 的可回写的buffer的地址。这时caller负责通过在调用函数之前存储属性getter的结果来初始化buffer，并在return时通过从buffer load
并用最终值调用setter来写回属性。

Swift代码如下：
```
func inout(_ x: inout Int) {
  x = 1
}
```
对应SIL如下：
```
sil @inout : $(@inout Int) -> () {
entry(%x : $*Int):
  %1 = integer_literal $Int, 1
  store %1 to %x
  return
}
```


## Swift Method Calling Convention @convention(method)

当前，方法调用约定与单独（freestanding）参数函数的调用约定相同。方法都柯里化（Curried
）了，最外层的是self参数，其他方法参数在内层。
因此，self参数在最后传递：

```
struct Foo {
  func method(_ x:Int) -> Int {}
}

sil @Foo_method_1 : $((x : Int), @inout Foo) -> Int { ... }
```


## Witness Method Calling Convention @convention(witness_method)

Witness方法调用是指WitnessTable里的协议方法。它与方法调用约定相同，不同之处在于它对泛型类型参数的处理。对于non-witness方法，在machine层面传递类型参数元数据的约定可能仅依赖于函数签名的静态部分就够了，但因为witness必须支持对self类型的多态派发，因此传递与Self相关的元数据必须以尽可能抽象的方式。


## C Calling Convention @convention(c)

在Swift的C模块导入器中，C类型总是被SIL映射到Swift的trivial类型。 SIL不关心平台ABI关于indirect return，寄存器与堆栈传递等要求。SIL中的C函数的传递和return总是传值，而忽略平台的调用约定。

因此目前SIL和Swift不能调用具有不定长参数的C函数。

## Objective-C Calling Convention @convention(objc_method)

### Reference Counts

Objective-C方法使用与ARC Objective-C相同的参数和返回值ownership规则。Selector以及Objective-C定义的ns_consumed，ns_returns_retained等属性也都支持。

对@convention(block)值进行apply调用，并不会consume这个block。

### Method Currying
 
OC方法的参数在SIL里也会柯里化，包括self参数，就像原生Swift方法一样。

```
@objc class NSString {
  func stringByPaddingToLength(Int) withString(NSString) startingAtIndex(Int)
}

sil @NSString_stringByPaddingToLength_withString_startingAtIndex \
  : $((Int, NSString, Int), NSString)
```

在SIL IR层，self和_cmd作为方法调用前面的参数，都被抽离出来了，默认隐藏了。

## GOSSARY
fragile physical变量?

