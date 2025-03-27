#glibc #Breakdown #thread #pthread



```c
struct pthread *pd = NULL;
int err = allocate_stack(iattr, &pd, &stackaddr, &stacksize);
int retval = 0;

if (__glibc_unlikely(err != 0))
  {
    retval = err == ENOMEM ? EAGAIN : err;
    goto out;
  }
```

#### Explanation:

This allocates the stack and thread descriptor using [`allocate_stack`](allocate_stack_thread.md). If allocation fails, it translates the error and exits. The line:

```c
retval = err == ENOMEM ? EAGAIN : err;
```

converts an `ENOMEM` (out of memory) error to `EAGAIN`, which is more appropriate for retryable resource exhaustion in `pthread_create`, aligning with POSIX expectations.