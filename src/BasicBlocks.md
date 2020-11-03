## Basic Blocks

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
函数入口总是其函数体的第一个BB。



SIL中，BB里包含了参数，这类似于LLVM的phi节点。BB的实参由其分支的前序bb进行绑定。


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

如果当前BB没有前序的BB，那么它的实参由调用者绑定：

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


## Debug Information

```
sil-scope-ref ::= 'scope' [0-9]+
sil-scope ::= 'sil_scope' [0-9]+ '{'
                 sil-loc
                 'parent' scope-parent
                 ('inlined_at' sil-scope-ref)?
              '}'
scope-parent ::= sil-function-name ':' sil-type
scope-parent ::= sil-scope-ref
sil-loc ::= 'loc' string-literal ':' [0-9]+ ':' [0-9]+
```

每条指令都可有一个debug的位置信息和一个SIL scope引用信息在最后。Debug的位置信息包括了文件名，行号，列号。如果debug位置信息是自动输出的，那么它的默认位置是SIL源文件。SIL Scope信息描述了生成SIL指令的Swift表达式最初在词法scope结构内的位置。SIL scopes同时保留了inline的信息.

## Declaration References

```
sil-decl-ref ::= '#' sil-identifier ('.' sil-identifier)* sil-decl-subref?
sil-decl-subref ::= '!' sil-decl-subref-part ('.' sil-decl-lang)? ('.' sil-decl-autodiff)?
sil-decl-subref ::= '!' sil-decl-lang
sil-decl-subref ::= '!' sil-decl-autodiff
sil-decl-subref-part ::= 'getter'
sil-decl-subref-part ::= 'setter'
sil-decl-subref-part ::= 'allocator'
sil-decl-subref-part ::= 'initializer'
sil-decl-subref-part ::= 'enumelt'
sil-decl-subref-part ::= 'destroyer'
sil-decl-subref-part ::= 'deallocator'
sil-decl-subref-part ::= 'globalaccessor'
sil-decl-subref-part ::= 'ivardestroyer'
sil-decl-subref-part ::= 'ivarinitializer'
sil-decl-subref-part ::= 'defaultarg' '.' [0-9]+
sil-decl-lang ::= 'foreign'
sil-decl-autodiff ::= sil-decl-autodiff-kind '.' sil-decl-autodiff-indices
sil-decl-autodiff-kind ::= 'jvp'
sil-decl-autodiff-kind ::= 'vjp'
sil-decl-autodiff-indices ::= [SU]+
```

有些SIL指令需要直接引用一些Swift声明。这些引用以#符号开头，后接限定的Swift声明的名称。有些Swift声明被拆解成不同的实体，分布在不同的SIL level上。 它们具有以下特征：限定名称以！开头，并且用一到多个.符号进行实体分割：

- getter: the getter function for a var declaration
- setter: the setter function for a var declaration
- allocator: a struct or enum constructor, or a class's allocating constructor
- initializer: a class's initializing constructor
- enumelt: a member of a enum type.
- - destroyer: a class's destroying destructor
- deallocator: a class's deallocating destructor
- globalaccessor: the addressor function for a global variable
- ivardestroyer: a class's ivar destroyer
- ivarinitializer: a class's ivar initializer
- defaultarg.n: the default argument-generating function for the n-th argument of a Swift func
- foreign: a specific entry point for C/Objective-C interoperability