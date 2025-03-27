#glibc #Breakdown #thread #pthread

```c
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

```
