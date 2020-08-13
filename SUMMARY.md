# Summary

* [Introduction](README.md)
* [0x0 Abstract](src/abstract.md)
* [0x1 SIL编译器](src/SILCompiler.md)
    * [0x11 SILGen]
    * [0x12 Guaranteed Optimization and Diagnostic Passes]
* [0x2 Syntax](src/Syntax.md)
    * [0x21 SIL Stage]
    * [0x22 SIL Types]
    * [0x23 Values and Operands]
    * [0x24 Functions]
    * [0x25 Basic Blocks]
    * [0x26 Debug Information]
    * [0x27 Declaration References]
    * [0x28 Linkage]
    * [0x29 VTables]
    * [0x2A Witness Tables]
    * [0x2B Default Witness Tables]
    * [0x2C Global Variables]
    * [0x2D Differentiability Witnesses]
* [0x3 Dataflow Errors](src/Dataflow.md)
    * [0x31 Definitive Initialization]
    * [0x32 Unreachable Control Flow]
    * [0x33 Values and Operands]
    * [0x34 Functions]
* [0x4 Ownership SSA](src/OwnershipSSA.md)
    * [0x41 Value Ownership Kind]
        * [0x42 Owned]
        * [0x43 Guaranteed]
        * [0x44 None]
        * [0x45 Unowned]
* [0x5 Runtime Failure](src/RuntimeFailure.md)
* [0x6 Undefined Behavior](src/UndefinedBehavior.md)
* [0x7 Calling Convention](src/SILStage.md)
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

