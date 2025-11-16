
glibc/[csu](csu%20folder.md)/libc-tls.c
```c
static void
init_static_tls (size_t memsz, size_t align)
{
  /* That is the size of the TLS memory for this object.  */
  GL(dl_tls_static_size) = roundup (memsz + GLRO(dl_tls_static_surplus),
				    TCB_ALIGNMENT);
#if TLS_TCB_AT_TP
  GL(dl_tls_static_size) += TLS_TCB_SIZE;
#endif
  GL(dl_tls_static_used) = memsz;
  /* The alignment requirement for the static TLS block.  */
  GL(dl_tls_static_align) = align;
  /* Number of elements in the static TLS block.  */
  GL(dl_tls_static_nelem) = GL(dl_tls_max_dtv_idx);
}

```