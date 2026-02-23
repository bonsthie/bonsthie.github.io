 ## Frontend
![[Block diagram of a CPU Frontend core.png]]

```mehrmaid
graph TD
  subgraph Phase1_The_Map_Cache
    A["Current PC (Instruction Pointer)"] --> B{"BTB (Branch Target Buffer)<br/>Have we been here before?"}
    B -->|Hit| B1["Predict exact target address instantly"]
    B -->|Miss| B2["Guess we just go to the next line"]
  end

  subgraph Phase2_The_Raw_Material_Cache
    B1 --> C{"L1 Instruction Cache (32KB)<br/>Are the raw bytes stored locally?"}
    B2 --> C
    C -->|Hit| D["Fetch bytes instantly at 32 bytes/cycle"]
    C -->|Miss| D_Miss["STALL: Wait for slow L2/L3 Cache or RAM"]
    D_Miss -.-> D
    D --> E["Pre-decode identifies instruction length"]
    E --> F["Instruction Queue"]
  end

  subgraph Phase3_The_Effort_Cache
    F --> G{"µop Cache (4k entries)<br/>Did we already decode this recently?"}
    G -->|Hit: The Fast Path| I["BYPASS DECODERS!<br/>Deliver 8 pre-built µops/cycle"]
    G -->|Miss: The Slow Path| J["6-way Decoder powers up<br/>Translates complex x86 to µops (6/cycle)"]
  end

  subgraph Phase4_Execution_Prep
    J --> U1["µop 1: Calc Address"]
    J --> U2["µop 2: Load Data"]
    J --> U3["µop 3: Add Math"]
    
    U1 --> Q["Instruction Decode Queue (144 µops)<br/>Buffer to keep Backend fed"]
    U2 --> Q
    U3 --> Q
    
    I -->|Direct Injection| Q
  end

  subgraph Phase5_Backend
    Q --> R["Rename & Allocate"]
    R --> S["Out-of-Order Execution Engine"]
  end
  
  %% Styling
  style B fill:#f9d0c4,stroke:#333,color:#000000
  style C fill:#f9d0c4,stroke:#333,color:#000000
  style G fill:#c4e1f9,stroke:#333,stroke-width:2px,color:#000000
  style I fill:#d4f9c4,stroke:#333,color:#000000
  style D_Miss fill:#f9c4c4,stroke:#333,stroke-dasharray: 5 5,color:#000000
```
> This graph is missing the MSROM and predecode link with the BPU, but I can't get this to render properly.
### Branch Prediction Unit (BPU)
- Predicts **next fetch address**
- Uses **BTB (12K entries)**:
    - branch PCs
    - target addresses
- Runs **every cycle**
- Drives instruction fetch
- Mispredict → pipeline flush (handled via MSROM)
### Instruction Fetch
- **L1 I-cache**:
    - 32 KB
    - fetches **32 bytes / cycle**
- SMT:
    - threads alternate → 32B every other cycle
- Fetch address from BPU
### Pre-decode
- Finds **instruction boundaries** (x86 = variable length, 1–15B)
- Detects branches
- Emits up to **6 macro-instructions / cycle**

### Instruction Queue
- Buffers macro-instructions
- Shared between SMT threads
- Performs **macro-op fusion**
    - two macros → one µop
    - saves backend bandwidth
### Decode
- **6-way decoder**
- Converts macro-ops → **fixed-length µops**
- Output goes to IDQ

### µop Cache (DSB)
- Caches **decoded µops**
- Checked **in parallel** with I-cache
- Capacity:
    - ~4K entries
    - **8 µops / cycle**
- Avoids:
    - pre-decode
    - decode
- Major frontend performance win

### MSROM (Microcode Sequencer / Microcode ROM)

MSROM is the **microcode execution engine inside the CPU**.  
It is entered when an instruction or event is **too complex, too rare, or too sensitive**
to be handled entirely by hardwired logic.

Its primary goal is **correctness and recoverability**, not performance.

---

#### What triggers MSROM

- Complex x86 instructions  
  - `REP MOVSB`, `REP STOSB` (`memcpy(dst, src, n)`)
  - `ENTER`, `BOUND` (`function prologue / bounds check`)
  - Atomic / system ops (`lock cmpxchg`)

- Assists (exceptional events)
  - TLB miss (`first access to mapped page`)
  - Page fault (`PTE invalid → #PF to OS`)
  - Branch misprediction recovery 
  - Machine clears (`self-modifying code`)
  - Floating-point denormals (`1e-310 * 2.0`)

- Security & compatibility handling
  - Speculation mitigations (`lfence after bounds check`)
  - Post-silicon microcode updates (`Spectre fix`)

#### How MSROM works

When entered, MSROM:
1. Temporarily takes control of the frontend
2. Executes a **microcode routine**
3. Emits µops directly into the backend
4. Restores architectural state and resumes normal execution

Throughput is limited (typically **≤ 4 µops / cycle**)/

### Instruction Decode Queue (IDQ)
- Boundary between **in-order frontend** and **OOO backend**
- Capacity:
    - 144 µops (single thread)
    - 72 µops per SMT thread

---
## ## **Out-of-order execution** 
![[Block diagram of a CPU OoO core.png]]
lets the CPU execute instructions as soon as their data is ready, not strictly in program order, to keep hardware busy and increase performance.

### Reorder Buffer (ROB)
- **512 entries**
- Core responsibilities:
    - register renaming
    - resource allocation
    - speculative tracking
    - in-order retirement
- Allocate:
    - **6 µops / cycle**
- Retire:
    - **8 µops / cycle**

### Register Renaming
- Architectural regs → **physical regs**
- Structures:
    - RAT (Register Alias Table)
    - PRF (Physical Register File)
- Separate PRFs:
    - INT
    - FP / SIMD

### Zero-latency idioms (Frontend / Rename-time)
- Zeroing idioms (`xor eax,eax`)
- Move elimination
- NOPs
- Some arithmetic bypasses  
    → save execution resources

### Scheduler / Reservation Station (RS)
- Tracks:
    - operand readiness
    - execution port availability
- Dispatch:
    - **6 µops / cycle**
- Smaller than ROB (~200 entries measured)

---
## Backend
![[Block diagram of a CPU backend core.png]]

### Execution Engine

#### INT / FP / Vector
- Ports: **0, 1, 5, 6, 10**
- ALU, LEA, shifts, MUL, FP, SIMD
- INT and FP/VEC use **separate PRFs**

#### Address Generation (AGU)
- Ports: **2, 3, 7, 8, 11**
- Required for **all loads and stores**
#### Stores
- Ports: **4, 9**
- STD = store data

### Port pressure matters
- Some ops are **port-restricted**
- Example:
    - FP divide → port 0 only
- Conflicts → dispatch stalls

### Load-Store Unit (LSU)
### Load capabilities
- Up to:
    - **3 loads / cycle**
    - (3×256-bit or 2×512-bit)
- Requires AGU

### Store capabilities
- Up to:
    - **2 stores / cycle**
    - (2×256-bit or 1×512-bit)

### Load / Store Buffers
- Load Buffer (Load Queue)
- Store Buffer (Store Queue)
- Track in-flight memory ops
- Enable:
    - store-to-load forwarding
    - memory reordering

### L1 Data Cache access

- **48 KB L1 D-cache**
- Load path:
    - cache lookup + TLB lookup **in parallel**
- L1 hit → value forwarded
    
### Miss handling
- L1 miss:
    - check L2
    - allocate **Fill Buffer (FB)**
- 16 fill buffers
- Speculative L3 lookup in parallel
- Same-line loads share FB (“glued” loads)

### Store optimizations
- Write-allocate (default)
- Store combining
- Streaming stores (no read-for-ownership)
- Non-temporal stores:
    - bypass cache pollution

### Memory ordering
- Weakly ordered
- Loads can bypass:
    - older loads
    - some older stores
- Memory disambiguation predicts dependencies
- Mis-prediction → pipeline flush (very expensive)

## TLB Hierarchy (Golden Cove)

### L1 TLB
- ITLB:
    - 256 entries (4K pages → 1 MB reach)
- DTLB:
    - 96 entries (4K pages → 384 KB reach)

### L2 TLB (STLB)
- 2048 entries
- Shared I + D
- ~8 MB reach (4K pages)
### Page walk acceleration
- Hardware page walkers:
    - **4 concurrent walkers**
- Paging-Structure Caches:
    - cache upper page-table levels
    - reduce loads during page walk
- Page walk loads use normal cache hierarchy

## SMT resource sharing
- Shared:
    - caches
    - TLBs
    - execution units
- Partitioned / replicated:
    - IDQ
    - ROB
    - RAT
    - RS
    - Load / Store Buffers
    - PRF
