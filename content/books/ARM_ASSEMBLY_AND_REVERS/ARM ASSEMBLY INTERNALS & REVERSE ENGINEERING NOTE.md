#arm #note #book 

there is 3 profile of arm ship

## Base of the ARCH
A: "application" -> ex : phone, lapto, IOT
R: designed for hard real-time or safety-critical systems -> ex : medical, HDD controllers
M: microcontroller -> ex : embedded,  Bluetooth, GPS

- arm v7 -> aarch32
- arm v8 -> aarch64/aarch32
- arm v9 -> same as v8 but with Scalable Vector Extension v2 (SVE2), Transactional Memory Extension (TME), and Branch Record Buffer Extensions (BRBE).

aarch32 has two mode :
	- A32 (BASE) all instr are 32 bit 
	- T32 (Thumb) instruction are manly 16 but have been able to use 32 bit instr
aarch64 compatible with arm A32 and T32

### Diff between arch and micro-arch
- architecture is the behavioral description of a processor and defines compo-
nents like the instruction set
- micro-architecture defines how the processor is
built. This includes the number and sizes of the caches, which features are imple-
mented, the pipeline layout, and even how the memory system is implemented

## LEVEL
![[arm_arch_level.png]]
- EL0 random program
- EL1 kernel
- hypervisor (KVM)
- **EL3** is used by processors with **TrustZone** to **switch between secure and normal worlds** through the **Secure Monitor**.
### Trusted Zone
- Protects **sensitive data and code** even if the main OS is compromised.
- Needed for:
	- OS integrity checks
    - Fingerprint or credential management
    - Device encryption keys
    - Banking / DRM / secure messaging apps
- Separates **trusted (secure)** and **untrusted (normal)** execution to stop malware from accessing critical data.


![[arm_trusted_zone_level.png]]

- S-EL0 Trusted apps (TAs) → user-level secure apps
- S-EL1 TEE OS → trusted kernel
- S-EL2 Trusted drivers (TDs) → secure hardware access
- S-EL3 Secure Monitor → world switching and full device access

### Exeption Level
this is to call into the upper level of execution

vector base address register (VBAR) -> store the addres of the base vector table
#### What happens during an exception
1. CPU stops current execution.
2. It jumps to a vector table in the higher EL using the `VBAR_ELn` register.
3. Runs the exception handler (OS or firmware code).
4. When finished, it returns to the previous level with `eret`

#### How they are trigger
##### Synchronous → caused by code (SVC/HVC/SMC)
EL0 → SVC (supervisor call) → EL1 (OS)
EL1 → HVC (hypervisor call) → EL2 (Hypervisor)
EL2 → SMC (secure monitor call) → EL3 (Secure Monitor)

##### Asynchronous → caused externally IRQ (Interrupt Request)/FIQ (Fast Interrupt Request)
EL0 → IRQ/FIQ (hardware interrupt) → EL1 (OS)
EL1 → IRQ/FIQ (virtual or physical interrupt) → EL2 (Hypervisor)
EL2 → IRQ/FIQ (secure interrupt) → EL3 (Secure Monitor)

# Execution state
they exist to support both **old 32-bit** and **new 64-bit** programs on the same chip.

### AARCH64 Register
#### overview
X0–X30 → 64-bit general-purpose registers
W0–W30 → lower 32 bits of X0–X30
V0–V31 → 128-bit SIMD & floating-point registers
XZR/WZR → zero register (always reads 0, discards writes)
SP → stack pointer (separate copy per EL)

#### register roles
X0–X7  → function arguments and return value  
X8     → indirect result pointer  
X9–X15 → caller-saved temporaries (saved by caller)  
X16–X18 → intra-procedure temporaries (for immediate values)  
X19–X28 → callee-saved registers (saved/restored by callee)  
X29    → frame pointer (FP)  
X30    → link register (LR, return address)

#### special register
PC (Program Counter) → holds address of next instruction  
SP (Stack Pointer)    → points to current stack frame  
FP (Frame Pointer)    → links stack frames for backtrace  
LR (Link Register)    → return address after BL (branch-and-link)

on aarch32 they are map to simple register
FP -> r11
IP -> r12
SP -> r13
LR -> r14
PC -> r15

>`PC` isn’t directly addressable in AArch64 (unlike AArch32’s `r15`).  You use **PC-relative instructions** like `ADR` / `ADRP` instead.
#### System Registers
system flag that are accecyble only by the instruction `mrs` to read it and `msr` to write it

PSTATE → processor status flags (NZCV, interrupt masks, EL bits, etc.)
VBAR_ELn → vector table address for each EL
SPSR_ELn → saved program status on exception

#### PSTATE flags
N –> Negative
Z –> Zero
C –> Carry
V –> Overflow

### zerso register (xzr/wzr)
with the register seting a register to zero the optimal way is now implicit not like in x86 with `xor rax rax` here you can `mov x0 xzr`

XZR lets A64 reuse existing instructions instead of defining new ones.
Example:
```asm
cmp Xn, #11        → alias for
subs XZR, Xn, #11  → subtract & set flags, result discarded
```
