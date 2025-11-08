## CONTEXT

* **LLVMContext** — owns uniqued IR state (types, constants, metadata).
* **Module** — translation unit; holds globals, functions, data layout, target triple.
* **DataLayout** — target data sizes/alignments.
* **Triple** — parsed target triple (`arch-vendor-os[-env]`).
* **TargetLibraryInfo** — libc/libm builtins knowledge.
* **DiagnosticHandler / DiagnosticInfo** — optimization/codegen diagnostics.

## IR (Core)

### Types & Values

* **Type** (IntegerType, PointerType, FunctionType, StructType, ArrayType, VectorType)
* **Value**, **User**, **Use**
* **Constant**, **ConstantInt**, **ConstantFP**, **UndefValue**, **PoisonValue**
* **GlobalValue**, **Function**, **GlobalVariable**, **GlobalAlias**, **GlobalIFunc**
* **Argument**, **BasicBlock**, **Instruction**

### Instructions

* **BinaryOperator**, **CastInst**, **CmpInst**
* **CallInst**, **InvokeInst**, **ReturnInst**
* **AllocaInst**, **LoadInst**, **StoreInst**
* **GetElementPtrInst (GEP)**
* **BranchInst**, **SwitchInst**, **PHINode**, **SelectInst**
* **AtomicRMWInst**, **FenceInst**, **CmpXchgInst**
* **MemIntrinsic** (MemCpyInst, MemMoveInst, MemSetInst)

### Building & Utilities

* **IRBuilder<>** — instruction creation.
* **Attribute**, **AttributeList**, **CallingConv::ID**
* **InlineAsm**
* **InstVisitor<>**
* **CloneFunction**, **ValueToValueMapTy**
* **BasicBlockUtils** (SplitBlock, SplitEdge, etc.)

### Metadata & Debug Info

* **Metadata**, **MDNode**, **MDString**, **NamedMDNode**
* **DebugLoc**, **DILocation**
* **DIBuilder**, **DICompileUnit**, **DISubprogram**, **DIType**

### IR I/O

* **SMDiagnostic**, **SourceMgr**
* **parseIRFile / parseAssemblyInto**
* **raw\_ostream**, **raw\_fd\_ostream**
* **WriteBitcodeToFile**, **BitcodeReader**, **BitcodeModule**
* **VerifierPass**, `verifyModule`, `verifyFunction`

## ANALYSES (IR level)

* **DominatorTree**, **PostDominatorTree**
* **Loop**, **LoopInfo**
* **ScalarEvolution**
* **AssumptionCache**
* **TargetTransformInfo (TTI)**
* **AAResults**, **AAManager**
* **MemorySSA**
* **BlockFrequencyInfo (BFI)**, **BranchProbabilityInfo (BPI)**
* **LazyValueInfo**
* **OptimizationRemarkEmitter**, **ProfileSummaryInfo**

## PASS INFRA (new PM)

* **PassBuilder**
* **ModulePassManager**, **CGSCCPassManager**, **FunctionPassManager**, **LoopPassManager**
* **ModuleAnalysisManager**, **CGSCCAnalysisManager**, **FunctionAnalysisManager**, **LoopAnalysisManager**
* **PreservedAnalyses**, **`PassInfoMixin<T>`**, **AnalysisKey**
* **PassInstrumentation**, **PassInstrumentationCallbacks**
* **PrintModulePass**, **PrintFunctionPass**

## JIT (ORC)

* **LLJIT**, **LLLazyJIT**
* **JITTargetMachineBuilder**, **DataLayout**
* **ThreadSafeContext**, **ThreadSafeModule**
* **ExecutionSession**, **MangleAndInterner**

## CODEGEN BRIDGE (IR → Machine IR)

* **Target**, **TargetMachine**, **TargetOptions**
* **TargetPassConfig / GISelPassConfig**
* **TargetLowering**, **TargetInstrInfo**, **TargetRegisterInfo**, **TargetFrameLowering**
* **TargetLoweringObjectFile**

### SelectionDAG

* **SelectionDAG**, **SDNode**, **SDValue**, **EVT**
* **SelectionDAGISel**, **FastISel**
* **CCState**, **CCValAssign**

### GlobalISel

* **IRTranslator**
* **Legalizer**, **LegalizerInfo**
* **RegisterBankInfo**, **RegBankSelect**
* **InstructionSelector**
* **CallLowering**
* **MachineIRBuilder**

## MIR (Machine IR)

* **MachineModuleInfo**
* **MachineFunction**, **MachineBasicBlock**
* **MachineInstr**, **MachineOperand**, **MachineMemOperand**
* **MachineRegisterInfo**, **MachineFrameInfo**
* **SlotIndexes**, **VirtRegMap**, **LiveIntervals**
* **Register**, **TargetRegisterClass**, **RegisterBank**
* **MachineLoopInfo**, **MachineDominatorTree**
* **MachineBlockFrequencyInfo**
* **RegScavenger**, **MachineScheduler**
* **MIRPrinter**, **MIRParser**

## ASM/MC LAYER

* **MCContext**
* **MCInst**, **MCOperand**
* **MCRegisterInfo**, **MCInstrInfo**, **MCSubtargetInfo**, **MCAsmInfo**
* **MCObjectFileInfo**
* **MCStreamer** (MCObjectStreamer, MCAsmStreamer)
* **MCAsmBackend**, **MCCodeEmitter**
* **MCSymbol**, **MCSection**, **MCFixup**
* **MCAsmParser**, **MCTargetAsmParser**
* **AsmPrinter**

## OBJECT & DEBUG UTILITIES

* **ObjectFile** (ELFObjectFile, COFFObjectFile, MachOObjectFile)
* **SectionRef**, **SymbolRef**, **RelocationRef**
* **DWARFContext**, **DILineInfo**, **DIInliningInfo**
