#glibc #Breakdown #thread #pthread

> 🧵 **Summary:**  
> allocate the default pthread_attr if the setting is set to NULL

```c
  const struct pthread_attr *iattr = (struct pthread_attr *) attr;
  union pthread_attr_transparent default_attr;
  bool destroy_default_attr = false;
  bool c11 = (attr == ATTR_C11_THREAD);
  if (iattr == NULL || c11)
    {
      int ret = __pthread_getattr_default_np (&default_attr.external);
      if (ret != 0)
        return ret;
      destroy_default_attr = true;
      iattr = &default_attr.internal;
    }
```

I won't talk much about this part — this sets up the base attributes if not set. If the user passes `NULL` or a special marker (`ATTR_C11_THREAD`) to `pthread_create`, the library creates a temporary local copy of the default attributes using `__pthread_getattr_default_np()` and fills in the values. This guarantees that the thread always has a complete and valid configuration.

---

## `struct pthread_attr` — internal representation

```c
struct pthread_attr {
  struct sched_param schedparam;   // POSIX scheduling priority parameters
  int schedpolicy;                 // Scheduling policy (e.g., SCHED_FIFO, SCHED_OTHER)
  int flags;                       // Bit flags like PTHREAD_CREATE_DETACHED, etc.
  size_t guardsize;                // Size of the guard page (prevents stack overflow)
  void *stackaddr;                 // If set, user-defined stack address
  size_t stacksize;                // Stack size (if custom)
  struct pthread_attr_extension *extension; // Optional advanced settings
  void *unused;                    // Reserved for future use / alignment
};
```

This struct holds all the configuration fields a thread can have — including scheduling, stack settings, and optional extensions like signal masks and CPU affinity. The `extension` pointer is lazily allocated only when advanced settings are used.

---

## `pthread_attr_t` and internal unions

```c
union pthread_attr_t {
  char __size[__SIZEOF_PTHREAD_ATTR_T];
  long int __align;
};
```

This is the **public opaque type**. It’s just a fixed-size blob big enough to hold the internal struct. The user only sees this type, never the internal layout.

```c
union pthread_attr_transparent {
  pthread_attr_t external;
  struct pthread_attr internal;
};
```

This **internal trick** allows the library to convert between the opaque `pthread_attr_t` and the real struct using overlapping memory. It enables safe casting between the user-visible and internal representation.

---

## Temporary value for safe copying

In `__pthread_attr_copy()`, the following pattern ensures safety:

```c
/* Avoid overwriting *TARGET until all allocations have succeeded. */
union pthread_attr_transparent temp;
temp.external = *source;           // Shallow copy

temp.internal.extension = NULL;   // Prevent sharing extension pointer


/* code that copy the default attr into temp */

  if (ret != 0)
    {
      /* Deallocate because we have ownership.  */
      __pthread_attr_destroy (&temp.external);
      return ret;
    }

  /* Transfer ownership.  *target is not assumed to have been
     initialized.  */
  *target = temp.external;
  return 0;

```

This avoids writing into `*target` until all memory allocations (like `cpuset` and `sigmask`) have succeeded. If any step fails, the temp object can be safely destroyed without affecting the original or target. This ensures that the copy is deep and doesn’t create shared ownership of internal pointers like `extension`.

This technique protects against partial failures and maintains memory safety.




full code of the default attr setup

```c
union pthread_attr_t
{
  char __size[__SIZEOF_PTHREAD_ATTR_T];
  long int __align;
};


struct pthread_attr
{
  /* Scheduler parameters and priority.  */
  struct sched_param schedparam;
  int schedpolicy;
  /* Various flags like detachstate, scope, etc.  */
  int flags;
  /* Size of guard area.  */
  size_t guardsize;
  /* Stack handling.  */
  void *stackaddr;
  size_t stacksize;

  /* Allocated via a call to __pthread_attr_extension once needed.  */
  struct pthread_attr_extension *extension;
  void *unused;
};

union pthread_attr_transparent
{
  pthread_attr_t external;
  struct pthread_attr internal;
};

int
__pthread_attr_setaffinity_np (pthread_attr_t *attr, size_t cpusetsize,
			       const cpu_set_t *cpuset)
{
  struct pthread_attr *iattr;

  iattr = (struct pthread_attr *) attr;

  if (cpuset == NULL || cpusetsize == 0)
    {
      if (iattr->extension != NULL)
	{
	  free (iattr->extension->cpuset);
	  iattr->extension->cpuset = NULL;
	  iattr->extension->cpusetsize = 0;
	}
    }
  else
    {
      int ret = __pthread_attr_extension (iattr);
      if (ret != 0)
	return ret;

      if (iattr->extension->cpusetsize != cpusetsize)
	{
	  void *newp = realloc (iattr->extension->cpuset, cpusetsize);
	  if (newp == NULL)
	    return ENOMEM;

	  iattr->extension->cpuset = newp;
	  iattr->extension->cpusetsize = cpusetsize;
	}

      memcpy (iattr->extension->cpuset, cpuset, cpusetsize);
    }

  return 0;
}


int
__pthread_attr_setsigmask_internal (pthread_attr_t *attr,
                                    const sigset_t *sigmask)
{
  struct pthread_attr *iattr = (struct pthread_attr *) attr;

  if (sigmask == NULL)
    {
      /* Mark the signal mask as unset if it is present.  */
      if (iattr->extension != NULL)
        iattr->extension->sigmask_set = false;
      return 0;
    }

  int ret = __pthread_attr_extension (iattr);
  if (ret != 0)
    return ret;

  iattr->extension->sigmask = *sigmask;
  iattr->extension->sigmask_set = true;

  return 0;
}


__pthread_attr_destroy (pthread_attr_t *attr)
{
  struct pthread_attr *iattr;

  iattr = (struct pthread_attr *) attr;

#if SHLIB_COMPAT(libc, GLIBC_2_0, GLIBC_2_1)
  /* In old struct pthread_attr, the extension member is missing.  */
  if (__builtin_expect ((iattr->flags & ATTR_FLAG_OLDATTR), 0) == 0)
#endif
    {
      if (iattr->extension != NULL)
        {
          free (iattr->extension->cpuset);
          free (iattr->extension);
        }
    }

  return 0;
}
k

int
__pthread_attr_copy (pthread_attr_t *target, const pthread_attr_t *source)
{
  /* Avoid overwriting *TARGET until all allocations have
     succeeded.  */
  union pthread_attr_transparent temp;
  temp.external = *source;

  /* Force new allocation.  This function has full ownership of temp.  */
  temp.internal.extension = NULL;

  int ret = 0;

  struct pthread_attr *isource = (struct pthread_attr *) source;

  if (isource->extension != NULL)
    {
      /* Propagate affinity mask information.  */
      if (isource->extension->cpusetsize > 0)
        ret = __pthread_attr_setaffinity_np (&temp.external,
                                             isource->extension->cpusetsize,
                                             isource->extension->cpuset);

      /* Propagate the signal mask information.  */
      if (ret == 0 && isource->extension->sigmask_set)
        ret = __pthread_attr_setsigmask_internal ((pthread_attr_t *) &temp,
                                                  &isource->extension->sigmask);
    }

  if (ret != 0)
    {
      /* Deallocate because we have ownership.  */
      __pthread_attr_destroy (&temp.external);
      return ret;
    }

  /* Transfer ownership.  *target is not assumed to have been
     initialized.  */
  *target = temp.external;
  return 0;
}


___pthread_getattr_default_np (pthread_attr_t *out)
{
  lll_lock (__default_pthread_attr_lock, LLL_PRIVATE);
  int ret = __pthread_attr_copy (out, &__default_pthread_attr.external);
  lll_unlock (__default_pthread_attr_lock, LLL_PRIVATE);
  return ret;
}

int pthread_create (){

	...
  const struct pthread_attr *iattr = (struct pthread_attr *) attr;
  union pthread_attr_transparent default_attr;
  bool destroy_default_attr = false;
  bool c11 = (attr == ATTR_C11_THREAD);
  if (iattr == NULL || c11)
    {
      int ret = __pthread_getattr_default_np (&default_attr.external);
      if (ret != 0)
	return ret;
      destroy_default_attr = true;
      iattr = &default_attr.internal;
    }

	...
}

```