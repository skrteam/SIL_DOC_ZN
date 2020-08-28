# Basic Blocks

```
sil-basic-block ::= sil-label sil-instruction-def* sil-terminator
sil-label ::= sil-identifier ('(' sil-argument (',' sil-argument)* ')')? ':'
sil-value-ownership-kind ::= @owned
sil-value-ownership-kind ::= @guaranteed
sil-value-ownership-kind ::= @unowned
sil-argument ::= sil-value-name ':' sil-value-ownership-kind? sil-type

sil-instruction-result ::= sil-value-name
sil-instruction-result ::= '(' (sil-value-name (',' sil-value-name)*)? ')'
sil-instruction-source-info ::= (',' sil-scope-ref)? (',' sil-loc)?
sil-instruction-def ::=
  (sil-instruction-result '=')? sil-instruction sil-instruction-source-info
```

一个函数的body包含至少一个basic block,bb的数量是由函数的CFG(control-flow-graph)的节点数决定的。
每个BB包含至少一条指令，并且以一条terminator指令结束。
函数入口总是它的第一个BB。



BB里包含了参数，这类似于LLVM的phi节点。BB的参数与前序BB到当前BB的branch路径有关。


```
sil @iif : $(Builtin.Int1, Builtin.Int64, Builtin.Int64) -> Builtin.Int64 {
bb0(%cond : $Builtin.Int1, %ifTrue : $Builtin.Int64, %ifFalse : $Builtin.Int64):
  cond_br %cond : $Builtin.Int1, then, else
then:
  br finish(%ifTrue : $Builtin.Int64)
else:
  br finish(%ifFalse : $Builtin.Int64)
finish(%result : $Builtin.Int64):
  return %result : $Builtin.Int64
}
```

如果当前BB没有前序的BB，那么它的参数来自调用者：

```
sil @foo : $@convention(thin) (Int) -> Int {
bb0(%x : $Int):
  return %x : $Int
}

sil @bar : $@convention(thin) (Int, Int) -> () {
bb0(%x : $Int, %y : $Int):
  %foo = function_ref @foo
  %1 = apply %foo(%x) : $(Int) -> Int
  %2 = apply %foo(%y) : $(Int) -> Int
  %3 = tuple ()
  return %3 : $()
}
```

如果该函数应用了Ownership SSA的规范，那么参数会显示地添加一些规范的注解信息，用来描述参数的Ownership语义。

```
sil [ossa] @baz : $@convention(thin) (Int, @owned String, @guaranteed String, @unowned String) -> () {
bb0(%x : $Int, %y : @owned $String, %z : @guaranteed $String, %w : @unowned $String):
  ...
}
```

注意，以上代码里的第一个参数%x拥有一个隐式的@none ownership语义，这是由于Int属于trivial type，默认是none，是符合规范的。