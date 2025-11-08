# LLVM Exploration

This repository demonstrates three different ways to lower a simple C function into LLVM’s intermediate representations:

1. **Hand‑written LLVM IR** at 3 optimization levels
2. **Programmatic IR emission** using the LLVM C++ API (`IRBuilder`)  
3. **Programmatic MIR emission** using the LLVM Machine IR API (`MachineIRBuilder`)
4. **Pass Creatation** creation of simple pass like `Constant Folding`
5. **TablenGen** : creation of simple TableGen example
6. **backend** this is fully in the [bonsthie/llvm-project `H2BLB-custom-backend branch`](https://github.com/bonsthie/H2BLB-custom-backend)

# llvm ir note

## noundef 
If it's not marked `noundef`, LLVM must assume it might be garbage or poison — so it can’t optimize as much.

## dso_local
tell that the function can be use without dynamic linking
by default `static` are `dso_local` but a function can if not exported if you don't put `dso_local` a function is `dso_preemptable` by default to say that he can be dynamicly replace.

## phi node
pick the right value depending on wich block your comming from.

## tail
`tail` marks a function call for potential tail call optimization 
it's valid when it's the last action in a block before a `ret`, `br`, or `unreachable`, and doesn’t require preserving the stack frame.

## callee caller

Caller = the function making the call
Callee = the function being called

## LLVM UB-Enabling Flags Reference

- **nsw** (No Signed Wrap): signed integer overflow ⇒ undefined behavior (UB).
- **nuw** (No Unsigned Wrap): unsigned integer overflow ⇒ undefined behavior (UB).
- **fast** (Fast-math): floating-point operations may be reassociated, ignore NaNs/Infs, etc. ⇒ relaxed IEEE semantics (UB-style optimizations).
- **reassoc**: allow reassociation of floating-point operations (e.g., transform `(x + a) + b` to `x + (a + b)`).

> **Note**: Default semantics are two's-complement wraparound for integers and strict IEEE-754 for floating-point. These flags are only present in IR when explicitly emitted (e.g., via Clang flags such as `-fstrict-overflow` or `-ffast-math`).


### Querying Flags in LLVM

- **LLVM IR level**:
  - `Instruction::hasNoSignedWrap()`, `Instruction::hasNoUnsignedWrap()`, `Instruction::hasNoNaNs()`, etc.
- **Machine IR level**:
  - `MachineInstr::getFlag(MIFlag Flag)` with `MIFlag::NoSignedWrap`, `MIFlag::NoUnsignedWrap`, `MIFlag::FmNoNaNs`, etc.

## Type
Type syntax:
* Vector: `<N x Ty>`
* Array: `[N x Ty]`
* strcut : `%MyStruct = type { i32, float, [4 x i8], {Ty1 , Ty2}}`
* anonymous/unnamed sturct : `Struct: {Ty1, Ty2, ...}`

### bfloat16 / vs IEEE-754 half bit-f16
**bfloat**
*1-bit sign
*8-bit exponent
*7-bit mantissa

**IEEE-754 half bit-f16**
*1-bit sign
*5-bit exponent
*10-bit mantissa

### addrspace
In LLVM IR, address spaces let you distinguish different “kinds” of memory—global vs. local vs. GPU‐shared vs. constant, etc.—at the type level. A pointer type is always in some address space, and by default it uses address space 0.
```
i8* addrspace(1)
<T>* addrspace(N)
```
- 0	“Normal” host memory
- 1	Often “global” device memory
- 2	“Constant” memory (read-only)
- 3	GPU local/shared memory (CUDA)

### attribute
store attribute groups
```
define i32 @bar(ptr noalias nocapture %arg) #0 {
; ...
}
attributes #0 = { noinline noreturn }
attributes #1 = { "target-cpu"="x86-64" "tune-cpu"="sandybridge" }
```

### triple
```
target triple = "target-vendor-os[-environment]"
```

# mir note
In LLVM IR every basic block must end with exactly one terminator; in MIR a MachineBasicBlock may end with none (fall-through) or several (be careful with linear order)
Linear order matters: moving bb.3 between bb.1 and bb.2 without adding a terminator changes semantics due to fall-through.

 textual MIR (.mir) and it’s YAML-based.
```yaml
--- 
# optional IR section (can be dropped when shrinking)
... 
--- 
name: foo
tracksRegLiveness: true
body: |
  bb.0:
    successors: %bb.1
    %0:gpr32 = COPY %x0
    $eflags = implicit-def
    %1:gpr32 = ADDWrr %0, %2 :: (load 4 from %stack.0)
  bb.1:
    RET
```

* explicit reg : part of the operand (ex : `add %1, %2`)
* implicit reg are not part of the operand marked with the `implicit`/`implicit-def` key word (ex : `cmpeq %1, %2` -> result goes here `$eflags`)

* `%0:gpr32`: A virtual register. As soon as a register class is set (here, gpr32), a register is considered a regular virtual register.
* `%1:gpr32(s32)`: A virtual register with a scalar type of size 32-bit.
* `%2:_(p0)`: A generic virtual register with no register class or register bank and a pointer type to the address space 0.
* `%3:gprb(<2 x s16>)`: A generic virtual register mapped on the gprb register bank and with the <2 x s16> vector type.


## pipeline
```sh
LLVM IR (or .bc) ──select──► MIR (MachineInstr, CFG) ──lower/print──► MC (MCInst stream) ──encode──► machine code bytes (.o/.exe)
```
* **MIR / Machine*** = program-level machine IR with a real CFG: MachineFunction → MachineBasicBlock → MachineInstr → MachineOperand
* **MC** = instruction-level representation for `printing`/`encoding`: `MCInst` → `MCOperand` plus `descriptors`/`encoders`/`streamers`. just a stream of instructions that an assembler/encoder can turn into bytes (or disassemble back).
> note : Important correction: MC is not “an interpretation of bitcode.” Bitcode is the serialized form of LLVM IR (much earlier).

## MIR  MC Classes

**MIR Classes:**

1. `MachineFunction`
2. `MachineBasicBlock`
3. `MachineInstr`
4. `MachineOperand`
5. `MachineMemOperand`
6. `MachineRegisterInfo`
7. `LiveRegUnits`
8. `TargetRegisterInfo`
9. `TargetRegisterClass`

**MC Classes:**

1. `MCInst`
2. `MCOperand`
3. `MCRegisterInfo`
4. `MCInstrDesc`
5. `MCInstrInfo`
6. `MCAsmInfo`
7. `MCSubtargetInfo`


## constraint of an operant
Operand constraints are defined in the TableGen `.td` files for the target’s instructions.
They tell the backend what kind of register or value each operand can be, and sometimes how operands relate to each other.

* `Register class constraint` – "must be `GR32`" → forces 32-bit regs only.
* `Tied operand constraint` – "`Op0` = `Op1`" → the def reuses the same register as the first use.
* `Fixed register constraint` – "must be `$eax`" → hardcoded ABI/hardware reg.
* `Subregister constraint` – "must be a sub_32 of a 64-bit reg".

## class

* `MachineIRBuilder` higher level class for building mir
* `MachineInstrBuilder` low level class for building mir

## tablegen
reg hierarchy
* `SubRegIndex<size, Offset>` for partial regs
* `CoveredBySubRegs` flag
* Example RCs (Register ClasS) :
* `SINGLES` -> [f32]
* `DOUBLES` -> [f64]
* `QUADS` -> [f128]
Operand constrain
* Reg class -> must be in given RC

## yaml mir anatomy
* section `frameInfo`, `stack:`, `body:`, `machineFunctionInfo:`
* keep `successors:`, `liveins:`, terminators in sync
* mem ops `:: (load/ store size form %stack.N)` = MachineMemOperand

# Theorical note

## SSA 

`alloca` `load` `store` make the LLVM IR non-SSA because they allow reassigning values to the same memory location (i.e., variable). `mem2reg` and `createPromoteMemoryToRegisterPass`/`PromoteToReg` API` transform this in ssa form with register and phi node
`SSAUpdater` from `TransformUtils` lib.

## def-use use-def

* Def-Use	For a given definition, what are all the uses of that value
* Use-Def	For a given use, what is the definition it depends on

```llvm
define i32 @example() {
entry:
  %a = add i32 5, 2       ; [DEF a]
  br label %next

next:
  %b = add i32 %a, 3      ; [USE a] [DEF b]
  %c = mul i32 %b, 4      ; [USE b] [DEF c]
  ret i32 %c              ; [USE c]
}
```

## backedge
going back in the CFG (ex: loop going back in the CFG)

##  critical edge
An edge that goes from a block with multiple successors to a block with multiple predecessors. Such edges are “critical” for many optimizations and are typically split by inserting a new intermediate block.
in llvm check if a node is critical with `isCriticalEdge`

## Irreducible graph
A CFG is irreducible if it contains one or more loops with multiple distinct entry points, making it impossible to transform into a hierarchy of single-entry loops.

## proxy (cost model)
A proxy cost model is a cheap heuristic that estimates instruction “cost” by counting or lightly weighting IR ops, letting early passes make decisions without invoking the full, target‑accurate backend model.

## InstCombine 
Is LLVM’s peephole optimization pass that matches small instruction patterns and rewrites them into more efficient or canonical forms (e.g., turning b * 2 into b << 1).

## liveness 
Is about whether a value needs to be kept around for future use.

## hoisting
Move something up in the cfg (ex: moving a block outside of a loop)

## sinking
Opposite of `hoisting` (ex : def of a value outside a if block inside)

## Folding
Is the compile‑time evaluation of constant expressions, replacing operations on known values with their computed literal results to simplify the IR.

## loops

```llvm
define void @loop_example() {
entry:
  br label %preheader

preheader:                                    ; preheader: dominates header, not in loop
  %i = alloca i32
  store i32 0, i32* %i
  br label %header

header:                                       ; header: first block in loop, also exiting block (branches outside)
  %iv = phi i32 [ 0, %preheader ], [ %iv_next, %latch ]
  %cmp = icmp slt i32 %iv, 10
  br i1 %cmp, label %exiting, label %exitblock

exiting:                                      ; exiting block: has a successor (%exitblock) outside the loop
  ; --- loop body ---
  %tmp = mul i32 %iv, 2
  br label %latch

latch:                                        ; latch: back‑edge to header
  %iv_next = add nsw i32 %iv, 1
  br label %header

exitblock:                                   ; exit block: outside the loop
  ret void
}
```

## topological traversal of a CFG

* Depth-First Preorder: visit a block before any of its successors.
* Depth-First Postorder: visit all successors (and their descendants) before the block itself.
* Reverse Postorder: reverse of postorder; approximates a topological order in reducible CFGs.
* Breadth-First Traversal: visit blocks level by level (by “distance” from the entry).
* Topological Sort: true topological order on the DAG of SCCs (requires condensing loops first).

## use object and User object
A User is anything that holds operands (like an instruction), and a Use is one of those operand slots linking the User to a Value.

# LLVM lib/class

## TargetLibraryInfo
Target library info is used by the vectorize pass to know if the target has a vectorized version of the instruction, to vectorize this part of the code using
```cpp
using TargetLoweringInfo::isFunctionVectorizable(StringRef F, const ElementCount &VF)
```

## DataLayout
This class is useful to know information on structs, like its size, with
```cpp
DL.getTypeSizeInBits(Ty)
DL.getABITypeAlignment(Ty)
DL.getTypeAllocSize(Ty)
```

## BlockFrequencyInfo
This class provides execution-frequency estimates for LLVM IR basic blocks—useful for guiding IR-level optimizations (e.g., hot-path inlining or block placement)
```cpp
// Get the relative frequency of a specific BasicBlock
BlockFrequency BFI.getBlockFreq(const BasicBlock *BB);

// Get an estimated profile count (absolute hits) if profile data is available
std::optional<uint64_t> BFI.getBlockProfileCount(const BasicBlock *BB);
```
You can see the BlockFrequencyInfo class on the LLVM IR by using the following command:
```sh
opt -passes='print<block-freq>' input.ll
```


# PASS Manager

* **`-enable-new-pm` / `-disable-new-pm`**
  Toggle the new PassManager in `opt` and Clang. By default, the new PM is enabled in recent releases.

* **`-passes=<pipeline>`**
  Specify a custom pipeline of passes (new PM syntax). Example: `-passes=funcearly-cse,instcombine,gvn`.

* **`-legacy-pass-manager`**
  Force use of the legacy PassManager even if the new PM is available.

* **`-print-before-all` / `-print-after-all`**
  Print the IR before or after every pass to stdout for inspection.

* **`-debug-pass=<Structure|Arguments>`**
  Emit debug output about pass scheduling:

  * `Structure`: shows the pass hierarchy.
  * `Arguments`: lists pass command‑line arguments.

* **`-verify-each`**
  Run LLVM’s verifier after every pass to catch IR invariant violations early.

* **`-time-passes`**
  Measure and report the execution time of each pass.

* **`-stats`**
  Print summary statistics collected by passes (e.g., number of transformations).

* **`-opt-bisect-limit=<n>`**
  Perform a bisect on the pass pipeline to isolate a problematic pass by limiting the first N passes.

* **`-disable-output`**
  Suppress writing any transformed IR or bitcode to disk (useful when only debugging passes).

* **`-view-cfg-after=<PassName>` / `-view-cfg-before=<PassName>`**
  Launch Graphviz to display the CFG after or before a specific pass.

how to write a advenced `--passes` cmd:
```sh
opt --passes='function(print,consthoist,loop(print),
instcombine<use-loop-info;max-iterations=3>),globaldce' myinput.ll -S -o -
```
Function-scoped, identified by function(...):
* print: Given the scope, this is the registration name of PrintFunctionPass
* consthoist: The constant hoisting function pass
* Loop-scoped, identified by loop(...):
* print: Given the scope, PrintLoopPass
* instcombine: The instruction combiner pass instantiated with two specific arguments: use-loop-info and max-iterations=3
Module-scoped:
* globaldce: The global dead-code elimination pass

## print
cmd line
`opt --print-passes`
Print IR after passes:
* `Legacy PM: FunctionPass::print()`
* `New PM: use PreservedAnalyses::all().getChecker<PrinterPass>()`

## check
Checks IR correctness : `verifyModule()`

## key analysis pass
`DominatorTree`: For dominance queries (needed for SSA form, LICM, etc.)
`LoopInfo`: Loop structure of the IR
`BlockFrequencyInfo`: Heuristic on execution frequency (for hot path optimization)
`AAResults`:	Alias analysis results
`TargetTransformInfo`:	Target-dependent cost model (e.g., is mul cheaper than shl)
`ValueTracking`:	Constant folding, poison, undef analysis

new pass manager
```cpp

PreservedAnalyses MyPass::run(Function &F, FunctionAnalysisManager &AM) {
  auto &DT = AM.getResult<DominatorTreeAnalysis>(F);
  auto &LI = AM.getResult<LoopAnalysis>(F);
  auto &AA = AM.getResult<AAManager>(F);
  // Use DT, LI, AA for analysis
  return PreservedAnalyses::all();
}
```

## key passes
Key passes:
* `InstCombine`: constant folding, pattern simplification (always useful).
* `mem2reg`: converts allocas to SSA form (necessary for many optimizations).
* `LCSSA`: prepares loops for safe transformations


# TablenGen
## warning 
the is multiple TablenGen compiler this change the TablenGen backend
* llvm-tablegen
* clang-tablegen
* mlir-tablegen

## Example
### input

```td
// Superclass with arguments and default fields
class BaseClass<string baseName, int baseValue> {
  string baseField = baseName;
  int    baseNum   = baseValue;
}

// Another superclass for demonstration
class Flags<int flagBits> {
  int bits = flagBits;
}

// Now a subclass using everything
class MyClass<string name, int size> : BaseClass<name, size>, Flags<size * 2> {
  string myName = name;
  int    mySize = size;
  bit    isActive = 1;
}
  ```
### result

```td
def Example {
  string baseField = "Widget";
  int    baseNum   = 8;
  int    bits      = 16;
  string myName    = "Widget";
  int    mySize    = 8;
  bit    isActive  = 1;
}
```

## Types

```
int : 64 bits int
bit : bool
bits<size> : sequence of 0 and 1
string : string
list<type> : table of type
dag : type is a special construct used to encode tree-like data structure
```

### dag
```td
class DAGHolder<dag P> {
  dag pat = P;
}

def WithDAG : DAGHolder<(mul $a, (add $b, $c))>;
```
you can type it too
```td
(add $lhs, $rhs)   // unnamed operands
(add GPR:$lhs, GPR:$rhs) // typed and named
```

## let
let : let you (aha) redefine a var in the next line `def` or in the next `scope {}`

## operator
! bang bang into the room !
 

# debug

## print cmd line
* `-print-before-all`
* `-print-after-all`
* `-print-before=PassName1[,PassName2]*`
* `-print-after=PassName1[,PassName2]*`

with clang u can use `-mllvm` to active opt command
```sh
clang -O3 -S input.c -o -mllvm -print-after-all
```

## print debug log
activate debug log of a pass

* `-debug` 
* `-debug-only=PassName1[,PassNAme2]*` 

exampple :
```sh
opt -passes=slp-vectorizer -debug-only=SLP input.ll -S –o output.ll
```

the debug name is set in the standard by de define `DEBUG_TYPE`

### how to setup debug
* include `llvm/Support/Debug.h`
* define `DEBUG_TYPE`
* use the `LLVM_DEBUG` macro to print

example :
```cpp
#include "llvm/Support/Debug.h"
#define DEBUG_TYPE "my-pass"
...
LLVM_DEBUG(dbgs() << "Processing: " << Instr << "\n");
```

## stats
statistics offer us a condensed way to know whether a pass was triggered 
```sh
clang -o out.s in.c -S -mllvm –stats
<snip>
    1 prologepilog - Number of functions seen in PEI
    2 regalloc - Number of copies coalesced
```

### setup stats

* include `llvm/ADT/Statistics.h`
* use the `STATISTIC` macro
    * NAME: The name of the variable for your counter.
    * DESC: The description of what this counter holds. This information will be printed in the final report


## llvm-extract
extract a function or a part of a function by keeping the code valid
```sh
llvm-extract -S -func=foo -o - input.ll # function
llvm-extract -S -func=foo:bb2 -o - input.ll # basic block
```

## llvm-reduce 
take a file in input a a small bash script and reduce the ir while the return of this bash script is still valid

has_bfi.sh
```sh
#!/bin/bash
llc $@ -o - | grep 'bfi'
exit $?
```

```sh
llvm-reduce --test=has_bfi.sh input.ll
```

old version of this is `bugpoint` (note the output of the sh script is the inverse of llvm-reduce)
```
bugpoint --compile-command=./has_not_bfi.sh --run-llc --compile-custom input.ll
```

## llvm-mc
use to test the mc layer

### mir
for mir reduce work but you can also
```sh
llc input.ll -simplify-mir
```

## exec a pass
```sh
llc --run-pass=opt input.mir -o -
```

## need a part on lldb but i'm to lazy right now

# Mir creation

## mir selection framework
this is for instruction selection (ISel)
* Selection DAG `SDIsel` (old SF)
    * monolitic one MachineFunctionPass
    * can only see one basic block at the time
* Fast `FIsel` (Fast derivation of SDIsel)
    * Fast
    * have access to the hole func but dangerous to look at more than one block because he can break the fallback to SDIsel if using value outside of the block
* GlobalIsel `GIsel` (new SF)
    * modular
    * can see through the whole func 


DAG is for SDIsel only the other work directly on the mir

* Use `EVT` when working in `SelectionDAG` (SDISel).
* Use `LLT` when working in `GlobalISel`.

## debug
`-stop-{before/after}`
* `-stop-after=irtranslator` → see the raw MIR after ABI lowering.
* `-stop-after=legalizer` → see legalized ops.
* `-stop-after=regbankselect` → see registers assigned.
* `-stop-after=isel`

# GIsel

```
LLVM IR
   ↓
IRTranslator (ABI lowering → G_* ops with LLT)
   ↓
Legalizer (make ops valid for target)
   ↓
RegBankSelect (assign regs)
   ↓
InstructionSelect (final target instructions)
   ↓
MachineInstr (ready for scheduling/regalloc)
```
