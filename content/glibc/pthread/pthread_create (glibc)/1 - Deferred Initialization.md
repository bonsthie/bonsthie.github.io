#glibc #Breakdown #thread #pthread

> 🧵 **Summary:**  
> Called once by the **first thread** to enable multi-threading support and set up signal handling for:
> 
> - [UID/GID](UID%20GID.md) synchronization using `SIGSETXID`
> - Thread cancellation (`pthread_cancel()`) using `SIGCANCEL`

This ensures both UID/GID changes and cancellation work correctly across all threads in the process.

```c
    /* Avoid a data race in the multi-threaded case, and call the
     deferred initialization only once.  */
  if (__libc_single_threaded_internal)
    {
      late_init ();
      __libc_single_threaded_internal = 0;
      /* __libc_single_threaded can be accessed through copy relocations, so
         it requires to update the external copy.  */
      __libc_single_threaded = 0;
    }

```

if `__libc_single_threaded_internal` is set to 1 this mean that the program is not set for thread so this will call `late_init()` to set up thread-related data structures, Enables thread-safe mechanisms and prepares TLS, signals, cancellation, etc. after that the `__libc_single_threaded_internal` is set to 0 to not call it next time and enable thread by setting `__libc_single_threaded` to 0

---

glibc/[nptl](ntpl%20folder)/pthread_create.c
```c
/* This performs the initialization necessary when going from
   single-threaded to multi-threaded mode for the first time.  */
static void
late_init (void)
{
  struct sigaction sa;
  __sigemptyset (&sa.sa_mask);

  /* Install the handle to change the threads' uid/gid.  Use
     SA_ONSTACK because the signal may be sent to threads that are
     running with custom stacks.  (This is less likely for
     SIGCANCEL.)  */
  sa.sa_sigaction = __nptl_setxid_sighandler;
  sa.sa_flags = SA_ONSTACK | SA_SIGINFO | SA_RESTART;
  (void) __libc_sigaction (SIGSETXID, &sa, NULL);

  /* The parent process might have left the signals blocked.  Just in
     case, unblock it.  We reuse the signal mask in the sigaction
     structure.  It is already cleared.  */
  __sigaddset (&sa.sa_mask, SIGCANCEL);
  __sigaddset (&sa.sa_mask, SIGSETXID);
  INTERNAL_SYSCALL_CALL (rt_sigprocmask, SIG_UNBLOCK, &sa.sa_mask,
			 NULL, __NSIG_BYTES);
}

```

---

```c
struct sigaction sa;
__sigemptyset (&sa.sa_mask);
```
- memset the signal struct

---
```c
/* The parent process might have left the signals blocked.  Just in
   case, unblock it.  We reuse the signal mask in the sigaction
   structure.  It is already cleared.  */
sa.sa_sigaction = __nptl_setxid_sighandler;
sa.sa_flags = SA_ONSTACK | SA_SIGINFO | SA_RESTART;
(void) __libc_sigaction (SIGSETXID, &sa, NULL);

__sigaddset(&sa.sa_mask, SIGCANCEL);
__sigaddset(&sa.sa_mask, SIGSETXID);
INTERNAL_SYSCALL_CALL(rt_sigprocmask, SIG_UNBLOCK, &sa.sa_mask,
                      NULL, __NSIG_BYTES);
```

> ⚠️ **Note:** This is only useful when using `setuid`, `setgid`, etc. — and can be ignored if you're not handling [UID/GID](UID%20GID.md) changes.

---

## 🔧 Purpose of This Code

This code installs a **signal handler** for the private internal signal `SIGSETXID`, used exclusively by glibc to synchronize **UID/GID changes** across threads in a multi-threaded program. It also ensures that `SIGSETXID` and `SIGCANCEL` are unblocked so that glibc's internal mechanisms function correctly.

---

## 📡 `SIGSETXID` Signal Setup

- **`SIGSETXID` is a glibc-internal signal**, not defined by the kernel.
    
- It is used when the process calls:
    
    - `setuid()`, `seteuid()`
    - `setgid()`, `setresgid()`
    - `setgroups()`

In a multi-threaded process, these syscalls must apply to **every thread**, not just the caller. So glibc sends `SIGSETXID` to all threads. Each thread handles it using the function:

👉 [`__nptl_setxid_sighandler`](__nptl_setxid_sighandler)

---

## ⚙️ Handler Configuration

- `sa.sa_sigaction = __nptl_setxid_sighandler;`
    
    - This tells glibc to run that function when a thread receives `SIGSETXID`
        
- `sa.sa_flags = SA_ONSTACK | SA_SIGINFO | SA_RESTART;`
    
    - `SA_ONSTACK`: Supports alternate signal stacks (important for threads)
        
    - `SA_SIGINFO`: Enables use of `siginfo_t` to pass metadata to the handler
        
    - `SA_RESTART`: Automatically retries interrupted syscalls
        
- `__libc_sigaction(SIGSETXID, &sa, NULL);`
    
    - Actually registers the handler for the signal
        

---

## 🚦 Signal Unblocking for Internal Threading Support

In some environments (or due to inheritance), signals like `SIGCANCEL` and `SIGSETXID` might be **blocked**. This code ensures they are unblocked:

- `SIGCANCEL` is used by glibc to implement `pthread_cancel()`
    
- `SIGSETXID` is used for UID/GID propagation
    

```c
__sigaddset(&sa.sa_mask, SIGCANCEL);
__sigaddset(&sa.sa_mask, SIGSETXID);
INTERNAL_SYSCALL_CALL(rt_sigprocmask, SIG_UNBLOCK, &sa.sa_mask,
                      NULL, __NSIG_BYTES);
```

- Ensures both signals are **delivered properly** to threads.
    

---

## 🧠 Why This Matters

Without this setup:

- UID/GID changes in multi-threaded programs would be incomplete
    
- `pthread_cancel()` might fail silently
    
- Some threads may retain old privileges, violating security expectations
    

> 🧠 This is one of the most subtle but critical parts of glibc's thread coordination mechanisms. If you don't use `setuid()` or `setgid()` — you can safely ignore this.