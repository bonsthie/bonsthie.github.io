#glibc #folder #explain
# 📁 What is the `csu` Folder in glibc?

The `csu` (C Startup) folder in the glibc source tree contains the low-level startup code that runs **before ****`main()`** in a C program. This code sets up the environment necessary for a C program to start correctly.

It plays a vital role in:

---

## 🚀 Responsibilities

- **C runtime startup sequence** (prepares arguments, environment, etc.)
    
- **Calling ****`main()`** safely after setup
    
- **Setting up ****`.init`**** and ****`.fini`**** functions** for constructors and destructors
    
- **Entry points for both static and dynamic binaries**
    
- **Setting up PIE (Position Independent Executable) vs non-PIE execution**
    

---

## 🧩 Key Files

- `crt1.c`, `crt1.o` – Startup code for dynamically linked executables
    
- `Scrt1.c`, `Scrt1.o` – Startup for statically linked executables
    
- `rcrt1.c` – For PIE executables
    
- `initfini.c` – Calls `.init` and `.fini` sections for constructor/destructor support
    
- `start.S` – Assembly entry point, very architecture-specific
    
- `dlstart.c` – Support for dynamic linker (`ld.so`) startup
    

---

## 🔧 How It Works

When you compile a C program, `crt1.o` or `Scrt1.o` is linked in by default. These startup files:

1. Set up the stack and registers (in assembly)
    
2. Call the C-level startup routine (`__libc_start_main`)
    
3. That function sets up things like:
    
    - Environment variables
        
    - [TLS](tls.md)
        
    - Standard I/O
        
    - Constructors (`__init_array__`, `.init`)
        
4. Finally, it calls your `main()` function
    
5. When `main()` returns, `exit()` is called
    

---