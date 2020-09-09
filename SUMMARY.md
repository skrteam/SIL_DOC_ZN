# Summary

* [Introduction](README.md)
* [0x0 摘要](src/abstract.md)
* [0x1 SIL和编译流程](src/SILCompiler.md)
    * [SILGen](src/SILCompiler.md#SILGen)
    * [Guaranteed Optimization and Diagnostic Passes](src/SILCompiler.md)
    * [General Optimization Passes](src/SILCompiler.md)
* [0x2 Syntax](src/Syntax.md)
    * [SIL Stage](src/SILStage.md)
    * [SIL Types](src/types/SILTypes.md)
    * [Values and Operands](src/ValuesAndOperands.md)
    * [Functions](src/Functions.md)
    * [Basic Blocks](src/BasicBlocks.md)
    * [Debug Information]
    * [Declaration References]
    * [Linkage](src/Linkage.md)
    * [VTables](src/VTable.md)
    * [Witness Tables]
    * [Default Witness Tables]
    * [Global Variables]
    * [Differentiability Witnesses]
* [0x3 Dataflow Errors](src/Dataflow.md)
    * [0x31 Definitive Initialization]
    * [0x32 Unreachable Control Flow]
* [0x4 Ownership SSA](src/OwnershipSSA.md)
* [0x5 Runtime Failure](src/RuntimeFailure.md)
* [0x6 Undefined Behavior](src/UndefinedBehavior.md)
* [0x7 Calling Convention](src/CallConvention.md)
    * [0x71 Swift Calling Convention @convention(swift)]
    * [0x72 Swift Method Calling Convention @convention(method)]
    * [0x73 Witness Method Calling Convention @convention(witness_method)]
    * [0x74 C Calling Convention @convention(c)]
    * [0x75 Objective-C Calling Convention @convention(objc_method)]
* [0x8 Type Based Alias Analysis](src/TBAA.md)
    * [0x81 Class TBAA]
    * [0x82 Typed Access TBAA]
* [0x9 Value Dependence](src/ValueDependence.md)
* [0xA Copy-on-Write Representation](src/Copy-on-Write.md)
* [0xB Instruction Set](src/Instructions/index.md)

