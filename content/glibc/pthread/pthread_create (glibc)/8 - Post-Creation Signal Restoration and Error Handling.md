#glibc #Breakdown #thread #pthread

```c
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

```

