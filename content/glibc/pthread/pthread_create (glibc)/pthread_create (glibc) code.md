#glibc #Breakdown #thread #pthread

glibc/[nptl](ntpl%20folder)/pthread_create.c

```c
int
__pthread_create_2_1 (pthread_t *newthread, const pthread_attr_t *attr,
		      void *(*start_routine) (void *), void *arg)
{
  void *stackaddr = NULL;
  size_t stacksize = 0;

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

  struct pthread *pd = NULL;
  int err = allocate_stack (iattr, &pd, &stackaddr, &stacksize);
  int retval = 0;

  if (__glibc_unlikely (err != 0))
    /* Something went wrong.  Maybe a parameter of the attributes is
       invalid or we could not allocate memory.  Note we have to
       translate error codes.  */
    {
      retval = err == ENOMEM ? EAGAIN : err;
      goto out;
    }


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

  /* Determine scheduling parameters for the thread.  */
  if (__builtin_expect ((iattr->flags & ATTR_FLAG_NOTINHERITSCHED) != 0, 0)
      && (iattr->flags & (ATTR_FLAG_SCHED_SET | ATTR_FLAG_POLICY_SET)) != 0)
    {
      /* Use the scheduling parameters the user provided.  */
      if (iattr->flags & ATTR_FLAG_POLICY_SET)
        {
          pd->schedpolicy = iattr->schedpolicy;
          pd->flags |= ATTR_FLAG_POLICY_SET;
        }
      if (iattr->flags & ATTR_FLAG_SCHED_SET)
        {
          /* The values were validated in pthread_attr_setschedparam.  */
          pd->schedparam = iattr->schedparam;
          pd->flags |= ATTR_FLAG_SCHED_SET;
        }

      if ((pd->flags & (ATTR_FLAG_SCHED_SET | ATTR_FLAG_POLICY_SET))
          != (ATTR_FLAG_SCHED_SET | ATTR_FLAG_POLICY_SET))
        collect_default_sched (pd);
    }

  if (__glibc_unlikely (__nptl_nthreads == 1))
    _IO_enable_locks ();

  /* Pass the descriptor to the caller.  */
  *newthread = (pthread_t) pd;

  LIBC_PROBE (pthread_create, 4, newthread, attr, start_routine, arg);

  /* One more thread.  We cannot have the thread do this itself, since it
     might exist but not have been scheduled yet by the time we've returned
     and need to check the value to behave correctly.  We must do it before
     creating the thread, in case it does get scheduled first and then
     might mistakenly think it was the only thread.  In the failure case,
     we momentarily store a false value; this doesn't matter because there
     is no kosher thing a signal handler interrupting us right here can do
     that cares whether the thread count is correct.  */
  atomic_fetch_add_relaxed (&__nptl_nthreads, 1);

  /* Our local value of stopped_start and thread_ran can be accessed at
     any time. The PD->stopped_start may only be accessed if we have
     ownership of PD (see CONCURRENCY NOTES above).  */
  bool stopped_start = false; bool thread_ran = false;

  /* Block all signals, so that the new thread starts out with
     signals disabled.  This avoids race conditions in the thread
     startup.  */
  internal_sigset_t original_sigmask;
  internal_signal_block_all (&original_sigmask);

  if (iattr->extension != NULL && iattr->extension->sigmask_set)
    /* Use the signal mask in the attribute.  The internal signals
       have already been filtered by the public
       pthread_attr_setsigmask_np interface.  */
    internal_sigset_from_sigset (&pd->sigmask, &iattr->extension->sigmask);
  else
    {
      /* Conceptually, the new thread needs to inherit the signal mask
	 of this thread.  Therefore, it needs to restore the saved
	 signal mask of this thread, so save it in the startup
	 information.  */
      pd->sigmask = original_sigmask;
      /* Reset the cancellation signal mask in case this thread is
	 running cancellation.  */
      internal_sigdelset (&pd->sigmask, SIGCANCEL);
    }

  /* Start the thread.  */
  if (__glibc_unlikely (report_thread_creation (pd)))
    {
      stopped_start = true;

      /* We always create the thread stopped at startup so we can
	 notify the debugger.  */
      retval = create_thread (pd, iattr, &stopped_start, stackaddr,
			      stacksize, &thread_ran);
      if (retval == 0)
	{
	  /* We retain ownership of PD until (a) (see CONCURRENCY NOTES
	     above).  */

	  /* Assert stopped_start is true in both our local copy and the
	     PD copy.  */
	  assert (stopped_start);
	  assert (pd->stopped_start);

	  /* Now fill in the information about the new thread in
	     the newly created thread's data structure.  We cannot let
	     the new thread do this since we don't know whether it was
	     already scheduled when we send the event.  */
	  pd->eventbuf.eventnum = TD_CREATE;
	  pd->eventbuf.eventdata = pd;

	  /* Enqueue the descriptor.  */
	  do
	    pd->nextevent = __nptl_last_event;
	  while (atomic_compare_and_exchange_bool_acq (&__nptl_last_event,
						       pd, pd->nextevent)
		 != 0);

	  /* Now call the function which signals the event.  See
	     CONCURRENCY NOTES for the nptl_db interface comments.  */
	  __nptl_create_event ();
	}
    }
  else
    retval = create_thread (pd, iattr, &stopped_start, stackaddr,
			    stacksize, &thread_ran);

  /* Return to the previous signal mask, after creating the new
     thread.  */
  internal_signal_restore_set (&original_sigmask);

  if (__glibc_unlikely (retval != 0))
    {
      if (thread_ran)
	/* State (c) and we not have PD ownership (see CONCURRENCY NOTES
	   above).  We can assert that STOPPED_START must have been true
	   because thread creation didn't fail, but thread attribute setting
	   did.  */
        {
	  assert (stopped_start);
	  /* Signal the created thread to release PD ownership and early
	     exit so it could be joined.  */
	  pd->setup_failed = 1;
	  lll_unlock (pd->lock, LLL_PRIVATE);

	  /* Similar to pthread_join, but since thread creation has failed at
	     startup there is no need to handle all the steps.  */
	  pid_t tid;
	  while ((tid = atomic_load_acquire (&pd->tid)) != 0)
	    __futex_abstimed_wait_cancelable64 ((unsigned int *) &pd->tid,
						tid, 0, NULL, LLL_SHARED);
        }

      /* State (c) or (d) and we have ownership of PD (see CONCURRENCY
	 NOTES above).  */

      /* Oops, we lied for a second.  */
      atomic_fetch_add_relaxed (&__nptl_nthreads, -1);

      /* Free the resources.  */
      __nptl_deallocate_stack (pd);

      /* We have to translate error codes.  */
      if (retval == ENOMEM)
	retval = EAGAIN;
    }
  else
    {
      /* We don't know if we have PD ownership.  Once we check the local
         stopped_start we'll know if we're in state (a) or (b) (see
	 CONCURRENCY NOTES above).  */
      if (stopped_start)
	/* State (a), we own PD. The thread blocked on this lock either
	   because we're doing TD_CREATE event reporting, or for some
	   other reason that create_thread chose.  Now let it run
	   free.  */
	lll_unlock (pd->lock, LLL_PRIVATE);

      /* We now have for sure more than one thread.  The main thread might
	 not yet have the flag set.  No need to set the global variable
	 again if this is what we use.  */
      THREAD_SETMEM (THREAD_SELF, header.multiple_threads, 1);
    }

 out:
  if (destroy_default_attr)
    __pthread_attr_destroy (&default_attr.external);

  return retval;
}
```