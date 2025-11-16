
# CURRENT
---
## [ARM ASSEMBLY INTERNALS & REVERSE ENGINEERING](https://www.oreilly.com/library/view/blue-fox/9781119745303/)
I started reading this book not from a reverse engineering perspective, but as a compiler engineer -- and honestly, there’s no better way to learn the architecture. This book goes into far more practical detail than the official ARM manuals, making complex topics much easier to internalize.

At first, I skimmed through the first three chapters, thinking I wouldn’t learn much. I was very surprised, they were packed with insights. The section on ELF was particularly good; even though I already know ELF well, this is the best-written explanation I’ve ever read on the topic.

The book also does an excellent job covering OS fundamentals and includes a great section on thread debugging, which helped me improve my own [thread debugging](thread%20debugging)

i not finish reading but you will be able to find my note [here](ARM%20ASSEMBLY%20INTERNALS%20&%20REVERSE%20ENGINEERING%20NOTE)

## [SSA BASE COMPILER DESIGN](https://link.springer.com/book/10.1007/978-3-030-80515-9)
i'm really at the start start of this book. This will be to learn in more detail about ssa and make the ssa form for the [scc](index#SCC%20--%20SIMPL%20C%20Compiler) compiler


# FINISH
---
## [LLVM CODE GENERATION](https://www.oreilly.com/library/view/llvm-code-generation/9781837637782/)
This book was really interesting and helped me solidify my understanding of LLVM’s middle and backend. I particularly appreciated that it covers not only code generation itself, but also topics like contributing to LLVM, understanding the folder architecture, and debugging.

From Chapter 9 onward, the book guides you through building a full backend from scratch, unlike the official LLVM tutorial, which suggests starting by copying an existing backend. Doing it from the ground up makes you face and understand design points that would otherwise remain hidden if you just copied existing code.

Before reading this book, I thought the IR was completely architecture-agnostic compared to the MIR, but that’s not true -- IR passes already take into account things like target sizes and calling conventions. You can even make IR pass specially for a architecture

you can find my note [here](LLVM%20CODE%20GENERATION%20NOTE)

## [# Intel Xeon Phi Processor High Performance Programming, 2nd Edition](https://www.oreilly.com/library/view/intel-xeon-phi/9780128091951/)
I read this book because we acceier 12 xeon phi node for a project of HPC that aim to run I read this book because we received 12 Xeon Phi nodes for an HPC project aimed at running optimized black hole simulations on this small cluster. Sadly, the project never saw the light of day -- but that didn’t stop me from diving into the architecture anyway!

This book was both interesting and frustrating. Sometimes it spends ten pages explaining very basic concepts, and then suddenly introduces an advanced one in just ten lines. Still, I really enjoyed the first part It was my first exposure to the concept of NUMA, which turns out to be _really_ important on a 64-core CPU.

One of my favorite facts from the book is that moving data from one core to another takes two cycles vertically and one cycle horizontally, a small but striking insight into the architecture.

I have to admit, I never finished it. The second half focuses more on HPC and distributed programming, while my main goal was to understand _CPU-level optimization_, not network-level scaling.

The hardware itself was a joy to study. The Xeon Phi’s design feels almost like a GPU, which makes sense since it evolved from one of Intel’s failed GPU projects. I especially liked learning that it included **16 GB of on-package VRAM** with different memory modes available.

And yes… I didn’t take any notes on the book. :sad emoji: