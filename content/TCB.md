# 🧩 What is the TCB (Thread Control Block)?

The **TCB** (Thread Control Block) is a small, per-thread data structure used internally by the runtime to store metadata about each thread.

It is typically located at the beginning or end of the thread's TLS (Thread-Local Storage) block.

---

## 📦 What’s Inside?

The exact contents are platform-specific, but usually include:

- A pointer to the DTV (Dynamic Thread Vector)
    
- `errno` (per-thread error value)
    
- Thread ID or pointer to the thread descriptor (`struct pthread`)
    
- Guards or padding for alignment
    

---

## 📍 Where is it?

- On **x86_64**, the TCB is typically placed at the end of the TLS block.
    
- The thread pointer (`TP`) register (e.g., `%fs`) points to the TCB.
    

### 🧭 Memory Layout Example

```text
mmap memory block (e.g., allocated with mmap or _dl_early_allocate)
0x00000000  ┌────────────────────────────────────────────┐
           │              Thread Stack                 │
           ├────────────────────────────────────────────┤
           │             TLS (.tdata/.tbss)            │
           ├────────────────────────────────────────────┤
0x00001230  │         TCB (Thread Control Block)        │ ← %fs points here (TLS_TCB_AT_TP)
           └────────────────────────────────────────────┘
```

> Note: On systems with `TLS_TCB_AT_TP`, the TCB is at the **end** of the TLS block, and the `%fs` register points to it directly.

---

## 🔧 Why It Matters

- Allows fast access to `__thread` variables
    
- Needed for per-thread libc functions (like `errno`)
    
- Fundamental for thread creation and context switching
    

---

## 🔍 Summary

The TCB is the anchor of thread-local state. It sits in TLS and is accessed via the thread pointer register. It's set up early by functions like `__libc_setup_tls`.