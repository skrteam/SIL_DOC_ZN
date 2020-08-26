# Values and Operands

```
sil-identifier ::= [A-Za-z_0-9]+
sil-value-name ::= '%' sil-identifier
sil-value ::= sil-value-name
sil-value ::= 'undef'
sil-operand ::= sil-value ':' sil-type
```

SIL中以$作为值的前缀。值的ID命名使用字母和数字。
也可能是'undef',如字面义。

和LLVM IR不同，把value作为操作数的指令只能接受value作为操作数。对于字面量常量，函数，全局变量和其他实体进行操作则需要专门的指令进行操组，如integer_literal, function_ref, global_addr, etc.