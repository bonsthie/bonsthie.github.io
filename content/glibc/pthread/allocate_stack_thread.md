> 🧵 **Summary:**  

you could find the full code [here](allocate_stack_thread_code)
## Base init

```c
  struct pthread *pd;
  size_t size;
  size_t pagesize_m1 = __getpagesize () - 1; // 4096
  size_t tls_static_size_for_stack = __nptl_tls_static_size_for_stack ();
  size_t tls_static_align_m1 = GLRO (dl_tls_static_align) - 1;

  assert (powerof2 (pagesize_m1 + 1));
  assert (TCB_ALIGNMENT >= STACK_ALIGN);
```

the `__getpagesize` on **x86_64** just return an hard coded value of **`4096`** i would of thought this was base on the [cpuid](https://fr.wikipedia.org/wiki/CPUID) 

the `__nptl_tls_static_size_for_stack()` is set at the end of the libc in the [`__libc_setup_tls()`](__libc_setup_tls.md) with the function [`init_static_tls()`](init_static_tls.md) this will init the [tls](tls)

---

## stack size

```c
/* Get the stack size from the attribute if it is set.  Otherwise we
   use the default we determined at start time.  */
if (attr->stacksize != 0)
  size = attr->stacksize;
else
  {
    lll_lock (__default_pthread_attr_lock, LLL_PRIVATE);
    size = __default_pthread_attr.internal.stacksize;
    lll_unlock (__default_pthread_attr_lock, LLL_PRIVATE);
  }
```

#### Explanation:

This snippet retrieves the desired stack size for a new thread:

- If the caller has set a custom stack size via `pthread_attr_setstacksize`, it uses that.
- Otherwise, it safely falls back to a global default.

> This fallback is mostly a safeguard: the default stack size is typically set by `init_pthread_attr()` or through the default `pthread_attr_t` object used by the main thread.

See [part 2 – Thread Attribute Handling](2%20-%20Thread%20Attribute%20Handling.md) for where the default is initialized.

---
## Get memory for the stack
>In this section, I’ll **only cover the standard path**—the `else` branch of the allocation logic. I’m skipping the `if (attr->flags & ATTR_FLAG_STACKADDR)` case for now, which handles **user-provided stacks**; I might revisit it later.
>
>**Note:** We'll assume `TLS_TCB_AT_TP`, which is the default for modern platforms like **x86_64** and **AArch64**. and other 64 byte system


memory plage




