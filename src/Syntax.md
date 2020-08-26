# 0x2 Syntax 句法


- SIL的句法是Swift句法的扩展，因为SIL依赖于Swift的类型系统以及声明(Declarations)。.sil文件就是Swift源码再加上扩展的SIL定义。
- 在句法解析过程中，仅解析Swift源码的各种声明；Swift的func函数体（body）部分（除了函数内声明的内部函数）以及顶层的代码会被SIL的句法解析忽略。
- .sil文件要求必须显式地引入外部依赖，包括标准库。

```
F.Y.I.
在Swift AST类型系统中，顶层的代码如果没有main函数，则会自动生成TopLevelCodeDecl的节点，并将顶层代码包裹其中。
```


以下是一个 .sil 文件的示例：
```

sil_stage canonical

import Swift

// Define types used by the SIL function.

struct Point {
  var x : Double
  var y : Double
}

class Button {
  func onClick()
  func onMouseDown()
  func onMouseUp()
}

// Declare a Swift function. The body is ignored by SIL.
func taxicabNorm(_ a:Point) -> Double {
  return a.x + a.y
}

// Define a SIL function.
// The name @_T5norms11taxicabNormfT1aV5norms5Point_Sd is the mangled name
// of the taxicabNorm Swift function.
sil @_T5norms11taxicabNormfT1aV5norms5Point_Sd : $(Point) -> Double {
bb0(%0 : $Point):
  // func Swift.+(Double, Double) -> Double
  %1 = function_ref @_Tsoi1pfTSdSd_Sd
  %2 = struct_extract %0 : $Point, #Point.x
  %3 = struct_extract %0 : $Point, #Point.y
  %4 = apply %1(%2, %3) : $(Double, Double) -> Double
  return %4 : Double
}

// Define a SIL vtable. This matches dynamically-dispatched method
// identifiers to their implementations for a known static class type.
sil_vtable Button {
  #Button.onClick: @_TC5norms6Button7onClickfS0_FT_T_
  #Button.onMouseDown: @_TC5norms6Button11onMouseDownfS0_FT_T_
  #Button.onMouseUp: @_TC5norms6Button9onMouseUpfS0_FT_T_
}

```

