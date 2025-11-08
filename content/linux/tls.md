# 🧵 What is TLS (Thread-Local Storage)?

TLS (Thread-Local Storage) is a mechanism that allows each thread in a multithreaded program to have its **own copy of a variable**, separate from other threads. This is essential for keeping thread-specific state like `errno`, or per-thread buffers. it is init in the gilibc in the function [`__libc_setup_tls`](__libc_setup_tls)

---

## 📦 Where is it Stored?

In [ELF](elf%20info.md) (Executable and Linkable Format) binaries, TLS data is stored in **two sections**:

- `.tdata`: contains **initialized** thread-local variables (e.g. `__thread int x = 5;`)
    
- `.tbss`: contains **uninitialized** thread-local variables (like a thread-local `__thread int y;` that defaults to zero)
    

Each thread gets its own copy of these sections when it's created.

---

## 🧠 How it Works

At program startup, glibc sets up TLS for the main thread. When a new thread is created (e.g. via `pthread_create`), glibc allocates memory for that thread's TLS by copying `.tdata` and zeroing `.tbss`.

Each thread has a special register pointing to its TLS block (e.g., `%fs` on x86_64), so accesses to `__thread` variables go to the correct location.


you can learn more about it [here](https://chao-tic.github.io/blog/2018/12/25/tls)