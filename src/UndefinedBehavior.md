# 0x6 Undefined Behavior

不正确地使用某些操作会导致未定义行为（Undefined Behavior）。
例如涉及Builtin.RawPointer的无效的未经检查的类型强转，或使用在LLVM level使用具有未定义行为的低至LLVM指令的编译器内置函数。具有未定义行为的SIL程序是无意义的，就像C中的未定义行为一样，并且没有可预测的语义。正确的Swift程序使用正确的标准库输出的有效SIL不应触发未定义行为，但不是所有情况都可以在SIL level得到诊断或验证。