glibc/[csu](csu%20folder.md)/libc-tls.c
```c
void
__libc_setup_tls (void)
{
  void *tlsblock;
  size_t memsz = 0;
  size_t filesz = 0;
  void *initimage = NULL;
  size_t align = 0;
  size_t max_align = TCB_ALIGNMENT; // x86_64 64
  size_t tcb_offset;
  const ElfW(Phdr) *phdr;

  struct link_map *main_map = GL(dl_ns)[LM_ID_BASE]._ns_loaded;

  __tls_pre_init_tp ();

  /* Look through the TLS segment if there is any.  */
  for (phdr = _dl_phdr; phdr < &_dl_phdr[_dl_phnum]; ++phdr)
    if (phdr->p_type == PT_TLS)
      {
	/* Remember the values we need.  */
	memsz = phdr->p_memsz;
	filesz = phdr->p_filesz;
	initimage = (void *) phdr->p_vaddr + main_map->l_addr;
	align = phdr->p_align;
	if (phdr->p_align > max_align)
	  max_align = phdr->p_align;
	break;
      }

  /* Calculate the size of the static TLS surplus, with 0 auditors.  */
  _dl_tls_static_surplus_init (0);

  /* We have to set up the TCB block which also (possibly) contains
     'errno'.  Therefore we avoid 'malloc' which might touch 'errno'.
     Instead we use 'sbrk' which would only uses 'errno' if it fails.
     In this case we are right away out of memory and the user gets
     what she/he deserves.  */
#if TLS_TCB_AT_TP
  /* Align the TCB offset to the maximum alignment, as
     _dl_allocate_tls_storage (in elf/dl-tls.c) does using __libc_memalign
     and dl_tls_static_align.  */
  tcb_offset = roundup (memsz + GLRO(dl_tls_static_surplus), max_align);
  tlsblock = _dl_early_allocate (tcb_offset + TLS_INIT_TCB_SIZE + max_align);
  if (tlsblock == NULL)
    _startup_fatal_tls_error ();
#elif TLS_DTV_AT_TP
  tcb_offset = roundup (TLS_INIT_TCB_SIZE, align ?: 1);
  tlsblock = _dl_early_allocate (tcb_offset + memsz + max_align
				 + TLS_PRE_TCB_SIZE
				 + GLRO(dl_tls_static_surplus));
  if (tlsblock == NULL)
    _startup_fatal_tls_error ();
  tlsblock += TLS_PRE_TCB_SIZE;
#else
  /* In case a model with a different layout for the TCB and DTV
     is defined add another #elif here and in the following #ifs.  */
# error "Either TLS_TCB_AT_TP or TLS_DTV_AT_TP must be defined"
#endif

  /* Align the TLS block.  */
  tlsblock = (void *) (((uintptr_t) tlsblock + max_align - 1)
		       & ~(max_align - 1));

  /* Initialize the dtv.  [0] is the length, [1] the generation counter.  */
  _dl_static_dtv[0].counter = (sizeof (_dl_static_dtv) / sizeof (_dl_static_dtv[0])) - 2;
  // _dl_static_dtv[1].counter = 0;		would be needed if not already done

  /* Initialize the TLS block.  */
#if TLS_TCB_AT_TP
  _dl_static_dtv[2].pointer.val = ((char *) tlsblock + tcb_offset
			       - roundup (memsz, align ?: 1));
  main_map->l_tls_offset = roundup (memsz, align ?: 1);
#elif TLS_DTV_AT_TP
  _dl_static_dtv[2].pointer.val = (char *) tlsblock + tcb_offset;
  main_map->l_tls_offset = tcb_offset;
#else
# error "Either TLS_TCB_AT_TP or TLS_DTV_AT_TP must be defined"
#endif
  _dl_static_dtv[2].pointer.to_free = NULL;
  /* sbrk gives us zero'd memory, so we don't need to clear the remainder.  */
  memcpy (_dl_static_dtv[2].pointer.val, initimage, filesz);

  /* Install the pointer to the dtv.  */

  /* Initialize the thread pointer.  */
#if TLS_TCB_AT_TP
  INSTALL_DTV ((char *) tlsblock + tcb_offset, _dl_static_dtv);

  call_tls_init_tp ((char *) tlsblock + tcb_offset);
#elif TLS_DTV_AT_TP
  INSTALL_DTV (tlsblock, _dl_static_dtv);
  call_tls_init_tp (tlsblock);
#endif

  /* Update the executable's link map with enough information to make
     the TLS routines happy.  */
  main_map->l_tls_align = align;
  main_map->l_tls_blocksize = memsz;
  main_map->l_tls_initimage = initimage;
  main_map->l_tls_initimage_size = filesz;
  main_map->l_tls_modid = 1;

  init_slotinfo ();
  /* static_slotinfo.slotinfo[1].gen = 0; -- Already zero.  */
  static_slotinfo.slotinfo[1].map = main_map;

  memsz = roundup (memsz, align ?: 1);

#if TLS_DTV_AT_TP
  memsz += tcb_offset;
#endif

  init_static_tls (memsz, MAX (TCB_ALIGNMENT, max_align));
}

```