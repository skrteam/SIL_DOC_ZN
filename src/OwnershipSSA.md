# 0x4 Ownership SSA

SILFunction如果被标记为[OSSA], 则表示该函数遵循Ownership SSA的形式。 OSSA是实参版本的SSA，通过给值操作边界上添加Ownship语义信息，从而确保实参的Ownership不变。
所有的SIL的值都被静态地赋予了一个Ownership语义。
所有对SIL值进行操作的SIL操作指令被分为两类：
- “normal uses”， 要求操作数，也就是操作的SIL值是有效（live）的
- “consuming uses”，消费该操作数，结束这个SIL值的生命周期，之后不可再使用该SIL值。

因此，所有的非消费类的操作指令都必须在“consuming uses”之前进行。
SIL值在所有可及的（见程序可及性）程序执行路径下，必须且仅被消费一次。
由此可以检查SIL值的内存泄漏和过度释放，野指针等问题。

看一个例子，注意定义和使用是成对出现的，并且用注释标明了：

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

可以注意到SIL中的每个值都是成对使用的（通过normal use和 consuming use）。如果违反了OSSA规则，会引发一个SILVerifier的错误，从而定位泄漏和野指针问题。


上面例子中只使用到了“Owned”语义，事实上，在SIL中，支持共四种Ownership语义：

- None 
  - 表示该值不需要进行内存管理，不用遵循OSSA的不变性的要求。例如：
    - trivial values (e.x.: Int, Float), 
    - non-payloaded cases of non-trivial enums (e.x.: Optional<T>.none)
    - all address types
- Owned
  - 该值具有独立的Ownership，不依附其他外部值。
  - 在可及的程序路径下只能销毁（通过destroy_value）或消费（通过apply,cast,store等对值重新绑定）一次。
- Guaranteed
  - 该值的生命周期不独立，依附于”Base Value“——其他owned或Guaranteed值。
  - 必须Guaranteed它的base value销毁/消费前，该值的有效性
  - begin_borrow指令开始，end_borrow结束
  - base value需要能静态地确保其值的有效性。
- Unowned
  - 值仅当前有效。使用的话，必须copy成一个Owned或者Guaranteed的新值。
  - 用于OC unsafe unowned 入参传递
  - 也用于，把trivial值按位转成non-trivial类型的值。
  - unowned值不可被consume


## Value Ownership Kind

### Owned

Owned所有权描述的是“仅作移动”的值。
Owned值在所有可及的程序路径下，需要且只能被消费一次。
IR检查器把始终未consume的Owned值标记为Leak；并且把多次consume的值标记未use-after-frees。
可以通过“forwarding uses”来建模move操作。这包括了cast  
和 transforming terminators(e.x.: switch_enum, checked_cast_br)。后者会转换输入值，在转化过程中consuming它，并生成一个转换好的owned的新值作为结果。


我们可以把每个owned SIL值当作“仅移动的值”，除非值被显式地进行copy_value。这样的话，ARC就可以只对特定的值进行语义上的操组，而且由于SILValue遵循SSA，因此ARC对象在声明生命周期中可以派生出多个独立的Owned SILValue，后者具有独立的生命周期和作用域，可以进行独立的检查。 


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

可以留意一下(%1)是如何被消费、转发、并且映射到新的值 (%2, %3, %5)上，而后者也都有独立的Owned值寿命。


### Guaranteed

Guaranteed值寿命依赖于“base value”。base value具有owned或者guaranteed属性。guaranteed值有效的前提是base value有效，这就要求在所有的范围下，base value寿命范围必须覆盖guaranteed值的。

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
> non-consuming的forwarding uses指令，输入值为base value，生成值为guaranteed的值。 此规则适用于递归的场景。

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

none ownership值被排除在OSSA之外，以昵称他们可以被当作普通的SSA来用，而不用担心破坏来OSSA 不变性的规则。但这并不意味着该代码就不违反其他的SIL规则（例如内存生命期不变性规则）。


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


## Forwarding Uses

SIL的部分子集指令根据操作数的值所有权类型定义其结果的值所有权类型。我们称这种指令为“forwarding指令”，任何使用该用户指令的用例我们称为“forwarding use”。这种推断通常发生在指令构造时，因此：
- 由于大多数情况下ownership是体现在指令中的，因此若通过程序操作forwarding指令，那么必须手动更新ownership forwarding。
- SIL文本并不能显式地表示出forwarding指令的所有权。相应的，指令的ownership是通常是通过解析后的操作元推断出来的。由于SILVerifier会在解析之后检查文本化SIL，因此你可以认为ownership的约束已经被正确地推断出来了。

根据构造操作数的值ownership类型不同以及返回类型不同，Forwarding的语义会有轻微的不同。
如下讨论：

- 如果操作数是@owned,forwarding指令会结束原操作数的生命，并生成一个新的值：对non-trivially类型生成@owned，对于trivially类型，则生成@owned。例如：用来表达类型转化的语义。

```
sil @unsafelyCastToSubClass : $@convention(thin) (@owned Klass) -> @owned SubKlass {
bb0(%0 : @owned $Klass): // %0 is defined here.

  // %0 is consumed here and can no longer be used after this point.
  // %1 is defined here and after this point must be used to access the object
  // passed in via %0.
  %1 = unchecked_ref_cast %0 : $Klass to $SubKlass

  // Then %1's lifetime ends here and we return the casted argument to our
  // caller as an @owned result.
  return %1 : $SubKlass
}
```

- 如果操作数是@guaranteed,那么forwaring指令针对non-trivially类型值会生成@guaranteed值，对于trivially类型值会生成@none值。对于non-trivially的例子，指令会隐式地对输入值开始一个新的begin borrow。由于这个borrow scope是隐式的，因此我们会像检查操作数一样检查结果的使用（支持递归）。这意味着对于任何guaranteed forwarded结果，我们不会看到对应的end_borrow指令。实际上end_borrow总是通过引入borrowed值的指令达成。一个guaranteed forwarding 指令的例子是struct_extract：

```
// In this function, I have a pair of Klasses and I want to grab some state
// and then call the hand off function for someone else to continue
// processing the pair.
sil @accessLHSStateAndHandOff : $@convention(thin) (@owned KlassPair) -> @owned State {
bb0(%0 : @owned $KlassPair): // %0 is defined here.

  // Begin the borrow scope for %0. We want to access %1's subfield in a
  // read only way that doesn't involve destructuring and extra copies. So
  // we construct a guaranteed scope here so we can safely use a
  // struct_extract.
  %1 = begin_borrow %0 : $KlassPair

  // Now we perform our struct_extract operation. This operation
  // structurally grabs a value out of a struct without safety relying on
  // the guaranteed ownership of its operand to know that %1 is live at all
  // use points of %2, its result.
  %2 = struct_extract %1 : $KlassPair, #KlassPair.lhs

  // Then grab the state from our left hand side klass and copy it so we
  // can pass off our klass pair to handOff for continued processing.
  %3 = ref_element_addr %2 : $Klass, #Klass.state
  %4 = load [copy] %3 : $*State

  // Now that we have finished accessing %1, we end the borrow scope for %1.
  end_borrow %1 : $KlassPair

  %handOff = function_ref @handOff : $@convention(thin) (@owned KlassPair) -> ()
  apply %handOff(%0) : $@convention(thin) (@owned KlassPair) -> ()

  return %4 : $State
}
```

- 如果操作数是@guaranteed,那么结果只必定是@none。
- 如果操作数是@unowned,那么结果只必定是@unowned。会像检查@unowned值一样检查该值，也就是确保先copy再使用。


这里的另一个问题是，即使绝大多数forwarding指令forwarding所有类型的ownership，但一般的情况并非如此。至于为什么，我们可以比较一下 struct_extract（不forward @owned ownership）和 unchecked_enum_data(forward所有ownership类型)。

不同的原因在于，struct_extract本质上只能提取较大对象的单个字段，这意味着该指令只能表示使用某个值的子字段（sub-field），而不是一次使用整个值。这违反了我们的约束，即，@owned值不能被部分consume：值要么完全live，要么完全dead。
另一方面，enum总是把元素装在一个tuple里来表示payload。这意味着使用unchecked_enum_data指令提取enum中的payload时可以consume这个那个enum和payload。


若想使用struct_extract来consume struct的话，我们可以使用destruct_struct指令，该指令一次consume整个struct，然后将struct的各个组成部分返还给用户。


```
struct KlassPair {
  var fieldOne: Klass
  var fieldTwo: Klass
}

sil @getFirstPairElt : $@convention(thin) (@owned KlassPair) -> @owned Klass {
bb0(%0 : @owned $KlassPair):
  // If we were to just do this directly and consume KlassPair to access
  // fieldOne... what would happen to fieldTwo? Would it be consumed?
  //
  // %1 = struct_extract %0 : $KlassPair, #KlassPair.fieldOne
  //
  // Instead we need to destructure to ensure we consume the entire owned value at once.
  (%1, %2) = destructure_struct $KlassPair

  // We only want to return %1, so we need to cleanup %2.
  destroy_value %2 : $Klass

  // Then return %1 to our caller
  return %1 : $Klass
}
```