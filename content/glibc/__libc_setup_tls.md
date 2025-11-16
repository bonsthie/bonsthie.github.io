> 🧵 **Summary:**  
> you could find the entire code [here](__libc_setup_tls_code.md)
> As the function name suggests, `__libc_setup_tls` initializes the **Thread-Local Storage ([TLS](tls.md))** for the main thread by:

- Searching for the TLS segment in the binary (via `.tdata` and `.tbss`) and extracting size/alignment info
    
- Allocating memory for the initial TLS block (including the [TCB](TCB.md) and alignment)
    
- Setting up global TLS metadata like `_dl_tls_static_size`, `_dl_tls_static_align`, and initializing the static DTV

note :

`dl_tls_static_align` is base on the TCB_ALIGNMENT but could be change if the [tls](tls.md) has a bigger alignment

the function use only brk for allocation to avoid using malloc because of errno who is a tls var


