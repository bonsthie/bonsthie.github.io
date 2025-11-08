# Hi,

### I'm Bonsthie -- a low-level enthusiast and compiler enjoyer!

I started coding 2 years ago and in the last 6 months I began focusing on compiler development with a focus on LLVM.

## What's here

On this website, you’ll find various things I’ve worked on -- mostly a place for me to store notes and write-ups.

- [Book readings and notes](books%20read.md)
- [My projects](#Main%20Work)
- [Write-ups](write%20ups.md)

# Main Work
---

## SIMPL project
---
This is a GitHub foundation I started with my friend [at0m741](https://github.com/at0m741/) that aims to build a low-level ecosystem: a **libc**, **compiler**, and **kernel**.
#### [SIMPL Libc](https://github.com/SIMPLproject/SLibc)

This is the most advanced part of the SIMPL project.  
Currently done:
- The library setup and build system are fully implemented, with symbol optimization inside the build.
- Simple string functions optimized for vectorization and IFUNC resolution.
- Basic `malloc` family functions.
- Some syscalls implemented.
- A simple `pthread` implementation.
#### [SIMPLV](https://github.com/SIMPLproject/SIMPLV)

This is a header-only vectorization library that uses the C preprocessor to help write vectorized code that is architecture-agnostic and size-agnostic (upscaling 128-bit vectors to 256-bit when appropriate).  
Currently it handles SSE and AVX -- it still needs to be extended to NEON intrinsics.  
This library exists mainly to simplify the libc code by providing a single implementation for vectorized algorithms. Of course this leads to slightly less aggressive hand-tuned optimizations than glibc, but for a function like `strlen` we reach ~90% of the glibc asm implementation's performance.

#### SCC -- SIMPL C Compiler

This is currently closed-source, but the name explains itself: it's the compiler of the SIMPL project, a C compiler that will target/use the SIMPL libc.

## LLVM work
---
#### [Clang walkthrough](clang_explain)
A write-up exploring the Clang main path to better understand how the Clang frontend works.

#### [LLVM tutorial backend](https://github.com/bonsthie/llvm-project/tree/H2BLB-custom-backend)
A backend implemented by following Quentin Collombet's _LLVM Code Generation_ book -- it helped me understand the LLVM backend flow.

#### Contributions
I contributed to the x86 GlobalISel backend in LLVM. I’m currently (on my free time -- which is limited) working on fixing issues around vector index instructions

## Ramdom
---
### [pthread_create explain](0%20-%20pthread_create%20(glibc)%20explain)
You can find my (incomplete) documentation of `pthread_create` (glibc implementation) here. This was done to locate a problem with my use of `clone3` in my own `pthread` implementation inside SIMPL libc.

