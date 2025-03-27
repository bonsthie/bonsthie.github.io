#glibc #Breakdown #thread #pthread

```c
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

```
