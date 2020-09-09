# 0x4 Ownership SSA

SILFountion如果被标记为[OSSA], 则表示该函数遵循Ownership SSA的形式。 OSSA是参数声明版本的SSA，通过给值和值操作添加相关Ownship所有权的语义信息，从而确保值的Ownership始终不变。
所有的SIL的值都被静态地赋予了一个Ownership语义。
所有对SIL值进行操作的SIL操作指令被分为两类：
- “normal uses”， 要求操作数，也就是操作的SIL值是有效的
- “consuming uses”，消费该操作数，结束这个SIL值的生命周期，之后不可再使用该SIL值。

因此，所有的非消费类的操作指令都必须在“consuming uses”之前进行。
SIL值在所有可及的（见程序可及性）程序执行路径下，必须且仅被消费一次。
由此可以检查SIL值的内存泄漏和过度释放，野指针等问题。

看一个例子：

```
sil @stash_and_cast : $@convention(thin) (@owned Klass) -> @owned SuperKlass {
bb0(%kls1 : @owned $Klass): // Definition of %kls1

  // "Normal Use" kls1.
  // Definition of %kls2.
  %kls2 = copy_value %kls1 : $Klass

  // "Consuming Use" of %kls2 to store it into a global. Stores in ossa are
  // consuming since memory is generally assumed to have "owned"
  // semantics. After this instruction executes, we can no longer use %kls2
  // without triggering an ownership violation.
  store %kls2 to [init] %globalMem : $*Klass

  // "Consuming Use" of %kls1.
  // Definition of %kls1Casted.
  %kls1Casted = upcast %kls1 : $Klass to $SuperKlass

  // "Consuming Use" of %kls1Casted
  return %kls1Casted : $SuperKlass
}
```

如果违反了OSSA规则，会英法一个SILVerifier的错误。

上面例子中只使用到了“Owned”语义，事实上，在SIL中，支持共四种Ownership语义：

- None 
  - 表示该值不需要进行内存管理，不遵循OSSA的不可变要求。例如：
    - trivial values (e.x.: Int, Float), 
    - non-payloaded cases of non-trivial enums (e.x.: Optional<T>.none)
    - all address types
- Owned
  - 该值具有独立的Ownership，不依附其他外部值。
  - 在可及的程序路径下只能销毁或消费一次。
- Guaranteed
  - 该值的Ownership不独立，生命周期依附于其他值——”Base Value“。
  - 必须Guaranteed它的base value销毁/消费前，该值的有效性
  - begin_borrow指令开始，end_borrow结束
- Unowned
  - 值仅当前有效。使用的话，必须copy一个Owned或者Guaranteed的新值。
  - 用于OC unsafe unowned 入参传递
  - 用于，当trivial值的类型转成non-trivial类型的新值，新值为unowned
  - unowned值不可被consume

## Value Ownership Kind

### Owned

Owned所有权描述的是“仅作移动”的值。
Owned值在所有可及的程序路径下，需要且只能被消费一次。
IR检查器把始终未consume的Owned值标记为Leak；并且把多次consume的值标记未use-after-frees。
可以通过“forwarding uses”来move Owned值。这包括了类型转换  
和 transforming terminators(e.x.: switch_enum, checked_cast_br)。后者会consume输入值，并生成一个新的owned值作为结果。

除非是被显式地进行copy_value，否则每个owned SIL值都会被高效地进行“移动”。这样的话，ARC就可以只对特定的值进行语义上的操组，而且由于SILValue遵循SSA，因此ARC对象的生命周期可以被拆分成独立的多个SILValue，对应独立的生命周期和作用域，进行独立的校验。 

例子如下：
```
// testcase.swift.
func doSomething(x : Klass) -> OtherKlass? {
  return x as? OtherKlass
}

// testcase.sil. A possible SILGen lowering
sil [ossa] @doSomething : $@convention(thin) (@guaranteed Klass) -> () {
bb0(%0 : @guaranteed Klass):
  // Definition of '%1'
  %1 = copy_value %0 : $Klass

  // Consume '%1'. This means '%1' can no longer be used after this point. We
  // rebind '%1' in the destination blocks (bbYes, bbNo).
  checked_cast_br %1 : $Klass to $OtherKlass, bbYes, bbNo

bbYes(%2 : @owned $OtherKlass): // On success, the checked_cast_br forwards
                                // '%1' into '%2' after casting to OtherKlass.

  // Forward '%2' into '%3'. '%2' can not be used past this point in the
  // function.
  %3 = enum $Optional<OtherKlass>, case #Optional.some!enumelt, %2 : $OtherKlass

  // Forward '%3' into the branch. '%3' can not be used past this point.
  br bbEpilog(%3 : $Optional<OtherKlass>)

bbNo(%3 : @owned $Klass): // On failure, since we consumed '%1' already, we
                          // return the original '%1' as a new value '%3'
                          // so we can use it below.
  // Actually destroy the underlying copy (``%1``) created by the copy_value
  // in bb0.
  destroy_value %3 : $Klass

  // We want to return nil here. So we create a new non-payloaded enum and
  // pass it off to bbEpilog.
  %4 = enum $Optional<OtherKlass>, case #Optional.none!enumelt
  br bbEpilog(%4 : $Optional<OtherKlass>)

bbEpilog(%5 : @owned $Optional<OtherKlass>):
  // Consumes '%5' to return to caller.
  return %5 : $Optional<OtherKlass>
}
```

可以留意一下(%1)是如何被消费、转发、并且映射到新的值 (%2, %3, %5)上，而后者也都有独立的生命周期。


### Guaranteed

Guaranteed值的生命周期依赖于“base value”。base value具有owned或者guaranteed所有属性。guaranteed值有效的前提是base value有效，这就要求在所有的范围下，base value的生命周期范围必须覆盖guaranteed值的。

这需要显式地使用 begin scope指令，如"begin_borrow","load_borrow"。并且需要成对地出现end scope指令： "end_borrow"

```
sil [ossa] @guaranteed_values : $@convention(thin) (@owned Klass) -> () {
bb0(%0 : @owned $Klass):
  %1 = begin_borrow %0 : $Klass
  cond_br ..., bb1, bb2

bb1:
  ...
  end_borrow %1 : $Klass
  destroy_value %0 : $Klass
  br bb3

bb2:
  ...
  end_borrow %1 : $Klass
  destroy_value %0 : $Klass
  br bb3

bb3:
  ...
}
```

可以注意到，%0的销毁指令总是在%1之后，所以编译优化器是无法在%1销毁前缩短%0的生命周期的。

对于具有guaranteed所有权属性的值，遵循以下数据流规则：
> non-consuming的forwarding uses指令，输入值为base value，生成值为guaranteed的值。 此规则适用于递归。

从而，新值依赖于原值的生命周期和对其对应的scope。
这么设计是为了这避免了生成过多幂等的scope（每个scope都需要一对指令生成）。

```
sil [ossa] @get_first_elt : $@convention(thin) (@guaranteed (String, String)) -> @owned String {
bb0(%0 : @guaranteed $(String, String)):
  // %1 is validated as if it was apart of %0 and does not need its own begin_borrow/end_borrow.
  %1 = tuple_extract %0 : $(String, String)
  // So this copy_value is treated as a use of %0.
  %2 = copy_value %1 : $String
  return %2 : $String
}
```

### None

None类型的Ownership表示该值不需要进行内存管理，OSSA机制需要忽略的部分。

包括如下类别：

- Trivial type (Int, Float, Double)
- Non-payloaded non-trivial enums
- Address types

```
F.Y.I.
对于Non-payloaded non-trivial enums, 和Swift对enum的实现有关。
对于没有payload的enum，最简化的实现版本类似C的enum，因此可以看作是一种特殊的Trivial Type
```

```
sil @none_values : $@convention(thin) (Int, @in Klass) -> Int {
bb0(%0 : $Int, %1 : $*Klass):

  // %0, %1 are normal SSA values that can be used anywhere in the function
  // without breaking Ownership SSA invariants. It could violate other
  // invariants if for instance, we load from %1 after we destroy the object
  // there.
  destroy_addr %1 : $*Klass

  // If uncommented, this would violate memory lifetime invariants due to
  // the ``destroy_addr %1`` above. But this would not violate the rules of
  // Ownership SSA since addresses exist outside of the guarantees of
  // Ownership SSA.
  //
  // %2 = load [take] %1 : $*Klass

  // I can return this object without worrying about needing to copy since
  // none objects can be arbitrarily returned.
  return %0 : $Int
}
```

### Unowned

适用于以下两个场景：

- ObjC函数调用规范中的入参。
  - 该规范要求调用者对入参先copy再使用。
  - SIL要求owned和guaranteed值作为入参传递前必须先copy。
  - 译者：如果同时遵循两个规范，那么需要在传参的前后copy两次，显然不合理。
  - 目前SIL还没有对这个场景专门设计一个机制处理，所以先用Unowned
- 从None Ownership的trivial type value 转过来的 non-trivial type value
  - 例如把一个trivial的unsafe指针值转成一个Class*的值。我们不能保证这个被指class对象的生命周期，所以我们必须先把这个值copy一下再用。
