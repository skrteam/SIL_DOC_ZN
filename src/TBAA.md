# 0x8 Type Based Alias Analysis

SIL支持两种基于类型的别名分析（TBAA）: Class TBAA 和 类型化访问的TBAA。


## Class TBAA

对Class实例和堆上其他的对象的引用的实现是通过指针，和SIL中的address类型不同，他们是SIL类型的一等公民(first-class)，指针可以被捕获(captured)和别名化(aliased)。
Swift是一个内存安全、静态类型的语言，所以Class的别名必须符合以下约束：

- Builtin.NativeObject类型可以作为堆上任意的Swift Native object的别名，包括swift class实例，通过alloc_box创建的box类型，或者thick的函数闭包。但不能作为ObjC对象的别名。
- AnyObject 和 Builtin.BridgeObject 可以作为任意Class实例的别名，包括OC和Swift都可以，但不包括堆上非Class实例。
- 两值Class类型的关联性
  - 如果两个值都是是同一个Class类型，可以互为别名；
  - 如果一值类型为$B,一值类型为$D,$B、$D是继承关系，则可以别名化；
  - 如果两值的Class类型不是关联的，则不能别名话。包括了类型特化了的泛型Class，例如 $C < Int >, $C < Float >
- 由于局部代码没有全局的可见性，因此必须假设泛型和协议的占位类型(泛型对应archetype, 协议对应 typealias)可能是任意的class实例的别名。即使从局部代码来看，某个class可能没有遵循特定协议，但在模块外可能它的某个extension实现了该协议。对于泛型，同理。

如果违反以上规则，在swift代码中解引用别名的引用值，那么程序可能会出现未定义的行为。
举个例子，__SwiftNativeNS[Array|Dictionary|String] classes 其实是 NS[Array|Dictionary|String] classes的别名，但他们在静态结构上是没有关联的。但是由于Swift不会对Foundation Class实例的存储类属性(StoredProperty，区别与计算型)进行直接的访问，因此对他们的别名化不会导致问题。

## Typed Access TBAA

对地址或引用的类型化访问，定义如下：
- 通过特定指令对特定地址的进行基于类型的读写访问，(e.x. load, store).
- 通过特定指令进行类型投射，给出特定类型内部元素指针偏移量(e.x. ref_element_addr, tuple_element_addr).

除了个别例外，对未关联对应类型的地址或引用进行typed访问，程序行为是未知的。

//??? 由此，优化器可以推断，对于两个Address，如果找不到一个archetype进行替换，使得替换后的两个类型的其中一个的类型正好是另一个的成员的类型，那么这两个Address也不能alias。
，如果不存在一个archetype，使得其中一个的类型和另一个的子成员的类型相同，那么对应的两个Address不可以alias。
以上规则同样适用于从类型投影（typed projection）获得的子类型。

看下面的例子：

```
struct Element {
  var i: Int
}
struct S1 {
  var elt: Element
}
struct S2 {
  var elt: Element
}
%adr1 = struct_element_addr %ptr1 : $*S1, #S.elt
%adr2 = struct_element_addr %ptr2 : $*S2, #S.elt
```

优化器判断%adr1 不能为 %adr2做别名，因为他们是从%ptr1(S1) 和 %ptr2(S2)里衍生出来的，S1和S2是不相关的类型。
然而在下面的例子里情况不同，%adr2是用一个类型转化得来的，而接下来的类型化操作都只和Element相关，%adr1对应S.slt: Element, %adr2对应$*Element, 因此他们是可能alias的。

```
%adr1 = struct_element_addr %ptr1 : $*S1, #S.elt
%adr2 = pointer_to_address %ptr2 : $Builtin.RawPointer to $*Element
```

typed TBAA也有例外，比如alias-introducing operations。这个机制允许类型双关(type-punning).目前唯一的例外的场景是pointer_to_address指令的非struct的一个变种。
优化器必须防御性地判断所有的alias-introducing operations都是address-roots。
Address Root是指生成address的指令，这类指令先于任意类型投影，索引，转换指令之前执行。以下都是有效的address roots：

- Object allocation, 比如 alloc_stack, alloc_box指令
- Address类型的函数入参。该场景不被认为是alias-introducing operations。SIL优化器从任意的address类型入参生成新的函数入参，这样做是非法的。因为这要求优化器能保证这个新的函数入参不是alias-introducing address root，且在函数调用规范中能很好地表示出来（规范对于地址类型，没有固定的表示）。
- pointer_to_address [strict]，即对一个没有类型化的指针应用严格的类型转化。pointer_to_address[strict]应用于 alias-introducing operation生成的值是非法的。目前，类型双关的地址只能是用非strict模式的pointer_to_address指令，从不透明指针(Opaque Pointer)生成。

使用 unchecked_addr_cast 指令进行 Address-to-address的类型转换，是可以透明地访问address root的，就像类型投影一样。

BasicBlock的Address类型入参会被认为是aliasing-introducing operations。

有一些指针的生成是天生就存在的，不需要作为TBAA的alias-introducing例外情况来考虑。例如，Builtin.inttoptr 会生成一个 Builtin.RawPointer类型的指针，可以任意地别名。类似的，LLVM内置的Builtin.bitcast 和 Builtin.trunc|sext|zextBitCast也不生成类型化的指针。
如果要对这些指针进行类型化的访问，必须先使用pointer_to_address指令进行转化。 而pointer_to_address是否是strict模式决定了是否可以别名。


内存可以被重新绑定到一个不相关的类型上。如果类型化访问的内存被绑定了具体类型，那么Address也可能为不关联的类型alias。因此，优化器也不能完全确定被访问的两个具有不相关类型的Address就不能alias。
例如，不能简单粗暴地消除指针比较，因为从这两个指针而来的两个Address可能在不同的程序执行点上被当作不相关的类型进行访问。


