
This is the signal handler that glibc uses to safely apply UID/GID changes across all threads in a multi-threaded process.

glibc/[nptl](ntpl%20folder)/ntpl_setxid.c
```c
/* Set by __nptl_setxid and used by __nptl_setxid_sighandler.  */
static struct xid_command *xidcmd;

/* We use the SIGSETXID signal in the setuid, setgid, etc. implementations to
   tell each thread to call the respective setxid syscall on itself.  This is
   the handler.  */
void
__nptl_setxid_sighandler (int sig, siginfo_t *si, void *ctx)
{
  int result;

  /* Safety check.  It would be possible to call this function for
     other signals and send a signal from another process.  This is not
     correct and might even be a security problem.  Try to catch as
     many incorrect invocations as possible.  */
  if (sig != SIGSETXID
      || si->si_pid != __getpid ()
      || si->si_code != SI_TKILL)
    return;

  result = INTERNAL_SYSCALL_NCS (xidcmd->syscall_no, 3, xidcmd->id[0],
				 xidcmd->id[1], xidcmd->id[2]);
  int error = 0;
  if (__glibc_unlikely (INTERNAL_SYSCALL_ERROR_P (result)))
    error = INTERNAL_SYSCALL_ERRNO (result);
  setxid_error (xidcmd, error);

  /* Reset the SETXID flag.  */
  struct pthread *self = THREAD_SELF;
  int flags, newval;
  do
    {
      flags = THREAD_GETMEM (self, cancelhandling);
      newval = THREAD_ATOMIC_CMPXCHG_VAL (self, cancelhandling,
					  flags & ~SETXID_BITMASK, flags);
    }
  while (flags != newval);

  /* And release the futex.  */
  self->setxid_futex = 1;
  futex_wake (&self->setxid_futex, 1, FUTEX_PRIVATE);

  if (atomic_fetch_add_relaxed (&xidcmd->cntr, -1) == 1)
    futex_wake ((unsigned int *) &xidcmd->cntr, 1, FUTEX_PRIVATE);
}
```


This function is a **signal handler** used internally by glibc to ensure that all threads apply a UID/GID change when a process transitions identity (e.g., after calling `setuid()`, `setgid()`).

It is installed by `late_init()` using `sigaction()` for the signal `SIGSETXID`.

---

## 🧩 Context: Why Is It Needed?

When a multi-threaded process changes its UID/GID:

- The **change must apply to all threads**, not just the caller.
    
- glibc uses the signal `SIGSETXID` to trigger a special handler in each thread.
    
- That handler is `__nptl_setxid_sighandler`, and it performs the required syscall.
    

---

## 🧠 Key Global Variable

```c
static struct xid_command *xidcmd;
```

- Shared command structure used by all threads
- Contains syscall number, arguments, error state, and a counter

---

## 📦 Structure: `struct xid_command`

glibc/[nptl](ntpl%20folder)/descr.h
```c
/* Opcodes and data types for communication with the signal handler to
   change user/group IDs.  */
struct xid_command
{
  int syscall_no;
  /* Enforce zero-extension for the pointer argument in

     int setgroups (size_t size, const gid_t *list);

     The kernel XID arguments are unsigned and do not require sign
     extension.  */
  unsigned long int id[3];
  volatile int cntr;
  volatile int error; /* -1: no call yet, 0: success seen, >0: error seen.  */
};
```

### ✅ Purpose:

- Acts as a **shared command buffer** for all threads to know:
    
    - What syscall to perform (e.g. `setresuid`)
    - What arguments to pass
    - How to synchronize completion ([`cntr`](cntr.md))
    - What error status occurred (if any)

---

## 🔍 Function Walkthrough

### 1️⃣ **Signal Validation Check**

```c
if (sig != SIGSETXID || si->si_pid != __getpid () || si->si_code != SI_TKILL)
  return;
```

- Ensures the handler only responds to correct internal signals
- Blocks signals from other processes (security)

---

### 2️⃣ **Perform the Syscall**

```c
result = INTERNAL_SYSCALL_NCS(xidcmd->syscall_no, 3,
                               xidcmd->id[0], xidcmd->id[1], xidcmd->id[2]);
```

- Executes the requested UID/GID change syscall (e.g., `setresuid`)
- Uses values from `xidcmd`

---

### 3️⃣ **Check for Errors**

```c
if (INTERNAL_SYSCALL_ERROR_P(result))
  error = INTERNAL_SYSCALL_ERRNO(result);
setxid_error(xidcmd, error);
```

- If syscall failed, record the error for the initiating thread to inspect

---

### 4️⃣ **Clear SETXID Flag**

```c
do {
  flags = THREAD_GETMEM(self, cancelhandling);
  newval = THREAD_ATOMIC_CMPXCHG_VAL(self, cancelhandling,
                                     flags & ~SETXID_BITMASK, flags);
} while (flags != newval);
```

- Clears a per-thread flag marking "setxid in progress"
- Uses atomic CAS to avoid races

---

### 5️⃣ **Notify This Thread is Done**

```c
self->setxid_futex = 1;
futex_wake(&self->setxid_futex, 1, FUTEX_PRIVATE);
```

- Thread may have been paused/waiting for UID change
- Notifies that it has completed the change

---

### 6️⃣ **Last Thread Wakes Initiator**

```c
if (atomic_fetch_add_relaxed(&xidcmd->cntr, -1) == 1)
  futex_wake((unsigned int *) &xidcmd->cntr, 1, FUTEX_PRIVATE);
```

- All threads decrement a shared counter
- The last one to finish wakes the original thread that called `setuid()`

---