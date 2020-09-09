# VTables

```
decl ::= sil-vtable
sil-vtable ::= 'sil_vtable' identifier '{' sil-vtable-entry* '}'

sil-vtable-entry ::= sil-decl-ref ':' sil-linkage? sil-function-name
```

SIL用  class_method, super_method, objc_method, and objc_super_method 指令来表示class方法的动态派发。

每个class类型都会有声明一个sil_vtable,可以在其中查找class_method 和 super_method。Vtable包含了class方法（包括继承的）到具体SIL函数实现的映射：

```
class A {
  func foo()
  func bar()
  func bas()
}

sil @A_foo : $@convention(thin) (@owned A) -> ()
sil @A_bar : $@convention(thin) (@owned A) -> ()
sil @A_bas : $@convention(thin) (@owned A) -> ()

sil_vtable A {
  #A.foo: @A_foo
  #A.bar: @A_bar
  #A.bas: @A_bas
}

class B : A {
  func bar()
}

sil @B_bar : $@convention(thin) (@owned B) -> ()

sil_vtable B {
  #A.foo: @A_foo
  #A.bar: @B_bar
  #A.bas: @A_bas
}

class C : B {
  func bas()
}

sil @C_bas : $@convention(thin) (@owned C) -> ()

sil_vtable C {
  #A.foo: @A_foo
  #A.bar: @B_bar
  #A.bas: @C_bas
}
```

可以注意到派生类C的A.bar是从派生类B继承而来的，因此vtable传递遵循最近可见性原则。Swift AST维护了这些覆写关系，从而可以查找派生类的覆写方法。

如果SIL函数是一个thunk函数的话，那么会根据函数的名字进行处理以找到链接的原始实现函数。

# Witness Tables
```
decl ::= sil-witness-table
sil-witness-table ::= 'sil_witness_table' sil-linkage?
                      normal-protocol-conformance '{' sil-witness-entry* '}'
```

SIL把泛型动态派发所需要的信息编码进WitnessTable。在生成二进制代码的时候，用这些信息来生成运行时方法派发表。这些信息也会被用于SIL优化，例如泛型函数特化。协议的每个显式声明遵循它的实例都会生成一个WitnessTable。泛型类型的所有实例会共享同一个泛型WitnessTable。派生类会从基类继承WitnessTable。


/// conformance, 对协议的遵循，比如 protocol witness for IGreeting.sayBye(_:) in conformance ClassB


```
protocol-conformance ::= normal-protocol-conformance
protocol-conformance ::= 'inherit' '(' protocol-conformance ')'
protocol-conformance ::= 'specialize' '<' substitution* '>'
                         '(' protocol-conformance ')'
protocol-conformance ::= 'dependent'
normal-protocol-conformance ::= identifier ':' identifier 'module' identifier
```

每个protocol conformance都会对应一个VitnessTable并且给它一个唯一标示的键。

-  一个normal protocol conformance描述了其(潜在的未绑定泛型unbound generic)类型，它所遵循的协议，以及所属模块。从而源码里的protocol confromance声明能一一对应上。
-  如果基类遵循了某协议，那么派生类将通过一个inherited protocol conformance继承，它只是简单地引用了基类的protocol conformance。
-  如果一个泛型的实例遵循了某协议，那么会得到一个specialized conformance，它可以给normal conformance绑定泛型参数的。


```
// FYI： witness table for unbound generic conformance
sil_witness_table hidden <M> C<M>: Greeting module wt3 {
  associated_type T: M
  method #Greeting.sayHi!1: <Self where Self : Greeting> (Self) -> (Self.T) -> () : @$s3wt31CCyxGAA8GreetingA2aEP5sayHiyy1TQzFTW	// protocol witness for Greeting.sayHi(_:) in conformance C<A>
  method #Greeting.sayBye!1: <Self where Self : Greeting> (Self) -> (Self.T) -> () : @$s3wt31CCyxGAA8GreetingA2aEP6sayByeyy1TQzFTW	// protocol witness for Greeting.sayBye(_:) in conformance C<A>
}
```

Witness tables只和normal conformance直接关联。Inherited 和 specialized conformances都是间接地引用normal conformance的witness table。

```
sil-witness-entry ::= 'base_protocol' identifier ':' protocol-conformance
sil-witness-entry ::= 'method' sil-decl-ref ':' sil-function-name
sil-witness-entry ::= 'associated_type' identifier
sil-witness-entry ::= 'associated_type_protocol'
                      '(' identifier ':' identifier ')' ':' protocol-conformance
```

Witness tables 由以下的入口(entry)构成:
   - Base protocol entries,通过提供对protocol conformance的引用，来实现witnessed protocol的继承关系。
   - Method entries，提供了协议里面声明的方法到对应方法实现的映射。witnessed protocol里的required方法必须对应到一个method entry。
   - Associated type entries，提供了协议里声明的associatedtype到具体的witness type的映射。需要注意的是，witness type是Swift语言层面的类型，而不是SIL类型的。witnessed protocol里的required associatedtype必须对应到一个Associated type entry。
   - Associated type protocol entries，提供了associatedtype要求的protocol 对应到 protocol conformance的映射。


## Default Witness Tables
```
decl ::= sil-default-witness-table
sil-default-witness-table ::= 'sil_default_witness_table'
                              identifier minimum-witness-table-size
                              '{' sil-default-witness-entry* '}'
minimum-witness-table-size ::= integer
```

SIL在default witness table里编码里弹性(默认实现是可选的)的默认实现。
如果满足以下条件的话，我们就说协议里的条款(requirement)有一个弹性的默认实现：
- 该requirement具有一个默认实现
- 该requirement要么是协议中的最后一个，要么其所有后续的requirement都具有弹性的默认实现

requirement集合以及其默认的实现存储在protocol的元数据中。

minimum witness table size就是不包括任何弹性默认实现的witness table的尺寸。

default witness table的实际尺寸等于 最小尺寸 + 默认requirements的尺寸，但不会超过最大尺寸。

在加载的时候，如果运行时发现witness table的尺寸达不到最大尺寸（也就是说缺少部分requirement），那么会拷贝一份新的witness table，并从default witness把缺失的部分填充拷贝填充进来。这就保证了调用者在witness table中总是可以找到预期数量的requirement。并且framework的作者可以添加新的requirements，而不用打断客户端的代码执行，前提是新的requirements也有弹性的默认实现。


Default witness table是协议自身来标示的。只有具有public可见性的协议需要default witness table。private和internal协议不会被模块外可见，因此添加新的requirement不会有弹性的问题。

```
sil-default-witness-entry ::= 'method' sil-decl-ref ':' sil-function-name
```

Default witness tables目前只包含一种entry：
    - Method entries，把协议的方法requirement映射到一个SIL函数，该函数实现了对所有witness类型都能匹配的方法。




## Global Variables

```
decl ::= sil-global-variable
static-initializer ::= '=' '{' sil-instruction-def* '}'
sil-global-variable ::= 'sil_global' sil-linkage identifier ':' sil-type
                           (static-initializer)?
```

SIL可以表示全局变量，通过alloc_global, global_addr 和 global_value指令进行访问。

全局变量可以有一个静态初始化器（initializer），但要求它的初始值由字面量(literals)组成。初始化方法用字面量列表和聚合指令来表示，其中最后一条指令是静态initializer的顶层值（top-level value）:

```
sil_global hidden @$S4test3varSiv : $Int {
  %0 = integer_literal $Builtin.Int64, 27
  %initval = struct $Int (%0 : $Builtin.Int64)
}
```

如果一个全局变量没有静态initializer，那么必须在首次访问之前使用alloc_global指令来申请存储。一旦全局存储初始化了，接下来使用global_addr指令来投影(project)该值。

如果静态initializer的最后一条指令是object指令，那么全局变量是一个静态初始化的对象。这种情况下，该全局变量不能作为左值，也就是说对这个对象的引用是不能修改的。于是，这个变量只能使用global_value访问，而不能通过global_addr访问。


## Differentiability Witnesses
可微分witness，特定场景才用得到，不翻译了。