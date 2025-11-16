#glibc #Breakdown #thread #pthread

```c
  /* Initialize the TCB.  All initializations with zero should be
     performed in 'get_cached_stack'.  This way we avoid doing this if
     the stack freshly allocated with 'mmap'.  */

#if TLS_TCB_AT_TP
  /* Reference to the TCB itself.  */
  pd->header.self = pd;

  /* Self-reference for TLS.  */
  pd->header.tcb = pd;
#endif

  /* Store the address of the start routine and the parameter.  Since
     we do not start the function directly the stillborn thread will
     get the information from its thread descriptor.  */
  pd->start_routine = start_routine;
  pd->arg = arg;
  pd->c11 = c11;

  /* Copy the thread attribute flags.  */
  struct pthread *self = THREAD_SELF;
  pd->flags = ((iattr->flags & ~(ATTR_FLAG_SCHED_SET | ATTR_FLAG_POLICY_SET))
	       | (self->flags & (ATTR_FLAG_SCHED_SET | ATTR_FLAG_POLICY_SET)));

  /* Inherit rseq registration state.  Without seccomp filters, rseq
     registration will either always fail or always succeed.  */
  if ((int) THREAD_GETMEM_VOLATILE (self, rseq_area.cpu_id) >= 0)
    pd->flags |= ATTR_FLAG_DO_RSEQ;

  /* Initialize the field for the ID of the thread which is waiting
     for us.  This is a self-reference in case the thread is created
     detached.  */
  pd->joinid = iattr->flags & ATTR_FLAG_DETACHSTATE ? pd : NULL;

  /* The debug events are inherited from the parent.  */
  pd->eventbuf = self->eventbuf;


  /* Copy the parent's scheduling parameters.  The flags will say what
     is valid and what is not.  */
  pd->schedpolicy = self->schedpolicy;
  pd->schedparam = self->schedparam;

  /* Copy the stack guard canary.  */
#ifdef THREAD_COPY_STACK_GUARD
  THREAD_COPY_STACK_GUARD (pd);
#endif

  /* Copy the pointer guard value.  */
#ifdef THREAD_COPY_POINTER_GUARD
  THREAD_COPY_POINTER_GUARD (pd);
#endif

  /* Setup tcbhead.  */
  tls_setup_tcbhead (pd);

  /* Verify the sysinfo bits were copied in allocate_stack if needed.  */
#ifdef NEED_DL_SYSINFO
  CHECK_THREAD_SYSINFO (pd);
#endif

```

