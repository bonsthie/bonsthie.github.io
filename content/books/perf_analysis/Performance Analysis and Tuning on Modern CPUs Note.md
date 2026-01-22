
# ARCH

## Dynamic Frequency Scaling (DFS)
Also called **Turbo mode**.  
Allows the CPU to temporarily increase its frequency above the base clock to boost performance.  
This can distort **benchmarks**, since the CPU does not run at a constant frequency, reducing result consistency and accuracy.

---
## Architecture models (high level)

### 1) **Register-based, load–store architectures**
- Arithmetic ops use **registers only**    
- Memory accessed **only** via explicit load/store    
- Easy pipelining, OoO-friendly, compiler-friendly
- **Most modern ISAs**

Examples: **ARM**, **RISC-V**
### 2) **Register-based, register–memory architectures**
- Arithmetic ops may use **register + memory operand**
- ISA is not load–store, but **microarchitecture behaves like one**
- Legacy-friendly, complex decoding

Example: **x86**
### Small remark: **Stack-based architectures**
- Operands implicit on a stack
- Compact encoding, poor for pipelining
- Common in VMs, not modern CPUs

Examples: **JVM**, **WebAssembly**

---
## Pipeline Hazards (Reminder)
- Break ideal pipelining → **stalls**
- Affect **throughput / effective frequency**
- Fully handled by **hardware**    
### Structural
- Resource conflict (same unit, same cycle)
- Mitigation: more hardware (ALUs, ports)
### Data
- **RAW**: true dependency → forwarding
- **WAR**: false dependency → register renaming
- **WAW**: false dependency → register renaming

### Control
- Branches / flow changes
- Mitigation: branch prediction, speculation

---
## **Out-of-order execution** 
lets the CPU execute instructions as soon as their data is ready, not strictly in program order, to keep hardware busy and increase performance.

---

## A **superscalar engine**
executes **multiple independent instructions per cycle** by issuing them to several execution units in parallel.

---
## Branch Prediction
- Correct prediction → high throughput
- Misprediction → expensive pipeline flush
- Uses **dynamic, adaptive predictors**
### Branch Types
- **Unconditional / direct**
    - Always taken
    - Single target
    - Easy to predict
- **Conditional**
    - Taken / not taken
    - Forward (if/else): often **not taken**
    - Backward (loops): often **taken**        
- **Indirect**
    - Multiple targets
    - From switch, function pointers, virtual calls
    - Returns are also indirect

---
## CAHCE
### Cache is **address-based**

- Cache does **not store variables**
- Stores **memory addresses**
- Any access is:
    - Address → set → tag match → hit/miss
- **cache line** (usually 64B)



---
# TESTING

## CI Perfomoramce

1. Set up the system under test
2. Run benchmark suite
3. Collect and report results
4. Detect performance changes
5. Alert on unexpected regressions or gains
6. Visualize results for human analysis

### CI Threshold
Performance tests in CI must define explicit **acceptable variation thresholds**.  
Unexpected performance gains in unrelated code paths often indicate **measurement errors or regressions**, not better code.

### [LUCI](https://chromium.googlesource.com/infra/infra/+/main/doc/users/index.md)
performance test suite of the chromium project

## Manual Testing

- Always compare **baseline vs modified**  
- Never trust **single-run** results    
- Run benchmarks **multiple times**
- Treat results as a **distribution**    
- Check **standard deviation** first    
    - High variance → results unreliable
- Do **not compute speedups** on noisy data
- Unexpected speedups often mean **measurement error**
- Use **visualization** to validate results    
- **Outliers matter** — don’t discard blindly
- Prefer **median / percentiles** if data is skewed
- Reduce noise **before** increasing sample count
- More samples ≠ better results if variance remains high

## Timer
- Long-running tests: use **standard library time functions**
    - Based on system (wall-clock) time
- Short / precise tests: use **CPU cycle counters**
    - x86: `RDTSC`/`RDTSCP` instruction
    - ARM: architectural timer sts: use **CPU cycle counters**
### X86 Time Reg

- **`RDTSC`** — Read Time-Stamp Counter
    - Reads the **TSC** (Time Stamp Counter)
    - Counts cycles since reset
- **`RDTSCP`** — Serializing variant of `RDTSC`
    - Ensures better ordering for measurements

**Notes**
- Modern CPUs use **invariant TSC**
-  TSC frequency is constant (independent of DFS / Turbo)
- Still affected by out-of-order execution → use serialization if needed
### ARM Time Reg
- **`CNTVCT_EL0`** — Virtual Count Register (most commonly used in user space)
- **`CNTPCT_EL0`** — Physical Count Register (more privileged)

Both count cycles of the **architectural timer**, not core frequency.

Typical choice:
- **User-space timing** → `CNTVCT_EL0` 
- **Kernel / low-level timing** → `CNTPCT_EL0`

(They are the ARM equivalent of x86 `RDTSC`, but with a fixed-frequency timer.)