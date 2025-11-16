**ELF File Structure: Segments and Sections**

An ELF (Executable and Linkable Format) file is composed a header, segments and sections, each serving a distinct purpose. not that i will use x86-64 some struct will differ in the x86 like the program header

## ELF file layout

```c
+--------------------+
| ELF Header         |  
+--------------------+
| Program Header     |  
+--------------------+
| Section 1          |  
+--------------------+
| Section 2          |  
+--------------------+
| ...                |  
+--------------------+
| Section Header     |  
+--------------------+
```

## ELF Header

The ELF header is at the top of the ELF file and contains metadata about the file's structure, type, and how it should be loaded into memory. The total size of the ELF header is **64 bytes**. It is composed of several fields that specify:

```c
typedef uint16_t Elf64_Half;    // 16-bit unsigned integer
typedef uint32_t Elf64_Word;    // 32-bit unsigned integer
typedef uint64_t Elf64_Addr;    // 64-bit unsigned integer (address)
typedef uint64_t Elf64_Off;     // 64-bit unsigned integer (offset)

typedef struct {
    unsigned char   e_ident[16];      // EI_NIDENT is usually defined as 16
    Elf64_Half      e_type;           // Object file type
    Elf64_Half      e_machine;        // Architecture
    Elf64_Word      e_version;        // Object file version
    Elf64_Addr      e_entry;          // Entry point virtual address
    Elf64_Off       e_phoff;          // Program header table file offset
    Elf64_Off       e_shoff;          // Section header table file offset
    Elf64_Word      e_flags;          // Processor-specific flags
    Elf64_Half      e_ehsize;         // ELF header size in bytes
    Elf64_Half      e_phentsize;      // Program header table entry size
    Elf64_Half      e_phnum;          // Number of program header entries
    Elf64_Half      e_shentsize;      // Section header table entry size
    Elf64_Half      e_shnum;          // Number of section header entries
    Elf64_Half      e_shstrndx;       // Section header string table index
} Elf64_Ehdr;

```

### `e_indent`

the e_indent could be seen as a struct that look like that

```c
typedef struct {
    char    ei_magic[4];    // Magic number (0x7F 'E' 'L' 'F')
    uint8_t ei_class;       // File class (none or 32-bit or 64-bit)
    uint8_t ei_data;        // Data encoding (01 little endian or 02 big endian)
    uint8_t ei_version;     // ELF version normaly always 1
    uint8_t ei_osabi;       // OS ABI (target operating system ABI) is a lot of the time set to 0 (none)
    uint8_t ei_abiversion;  // ABI version (almost never use)
    uint8_t ei_pad[7];      // Padding (should be all 0's)
} Elf64_EIdent;
```

### `e_type`

```c
typedef enum {
    ET_NONE   = 0,  // No file type
    ET_REL    = 1,  // Relocatable file
    ET_EXEC   = 2,  // Executable file
    ET_DYN    = 3,  // Shared object file
    ET_CORE   = 4,  // Core file
    ET_LOOS   = 0xFE00,  // Operating system-specific
    ET_HIOS   = 0xFEFF,  // Operating system-specific
    ET_LOPROC = 0xFF00,  // Processor-specific
    ET_HIPROC = 0xFFFF   // Processor-specific
} Elf64_EType;
```

### `e_machine`

```c
typedef enum {
    EM_NONE        = 0,      // No machine
    EM_M32         = 1,      // AT&T WE 32100
    EM_SPARC       = 2,      // SPARC
    EM_386         = 3,      // Intel 80386
    EM_68K         = 4,      // Motorola 68000
    EM_88K         = 5,      // Motorola 88000
    EM_IAMCU       = 6,      // Intel MCU
    EM_860         = 7,      // Intel 80860
    EM_MIPS        = 8,      // MIPS R3000
    EM_ARM         = 40,     // ARM
    EM_X86_64      = 62,     // x86-64
    EM_AARCH64     = 183,    // ARM 64-bit (AARCH64)
    EM_RISCV       = 243,    // RISC-V
    EM_LOPROC      = 0xFF00, // Processor-specific
    EM_HIPROC      = 0xFFFF  // Processor-specific
} Elf64_EMachine;
```

### `e_version`

is always set to 1

### `e_entry`

Entry point address for executable could be NULL if none

### `e_phoff` and `e_shoff`

tell respectively the offset where the program header start and the offset of the section header 

### `e_flags`

is not use in x86 and x86-64

- arm and mips
    
    ```c
    typedef enum {
        // ARM-specific flags
        EF_ARM_RELEXEC       = 0x01,       // Relocatable executable
        EF_ARM_HASENTRY      = 0x02,       // Entry point is present
        EF_ARM_INTERWORK     = 0x04,       // Interworking between ARM and Thumb
        EF_ARM_APCS_26       = 0x08,       // Use APCS/26 (obsolete)
        EF_ARM_APCS_FLOAT    = 0x10,       // Floating point support
        EF_ARM_PIC           = 0x20,       // Position-independent code
        EF_ARM_ALIGN8        = 0x40,       // 8-byte aligned data
        EF_ARM_NEW_ABI       = 0x80,       // New ABI (Application Binary Interface)
        EF_ARM_OLD_ABI       = 0x100,      // Old ABI
        EF_ARM_SOFT_FLOAT    = 0x200,      // Software floating point
        EF_ARM_VFP_FLOAT     = 0x400,      // VFP (Vector Floating Point) support
        EF_ARM_MAVERICK_FLOAT = 0x800,     // Maverick floating point
    
        // MIPS-specific flags
        EF_MIPS_NOREORDER    = 0x00000001, // No reorder
        EF_MIPS_PIC          = 0x00000002, // Position-independent code
        EF_MIPS_CPIC         = 0x00000004, // PIC using call stubs
        EF_MIPS_ABI2         = 0x00000020, // N32 ABI
        EF_MIPS_OPTIONS_FIRST = 0x00000080, // .options section first
        EF_MIPS_32BITMODE    = 0x00000100, // 32-bit addresses
        EF_MIPS_FP64         = 0x00000200, // 64-bit floating point
        EF_MIPS_NAN2008      = 0x00000400  // IEEE 754-2008 NaN encoding
    } Elf64_EFlags;
    
    ```
    

### `e_ehsize`

ELF header size to future prof the file format

### help

to see a simple human readable of a ELF you can use the cmd

```c
readelf -h realy_realy_realy_important_binary
```

# Segments vs Sections

- **Segments**: These are relevant during runtime. They define how and where parts of the program should be loaded into both virtual and physical memory.
- **Sections**: These are primarily used during the linking process, organizing code and data into meaningful units, like functions, global variables, and symbol tables.

### ELF Structure in an Executable

An executable typically contains three segments:

1. **Data Segment**: Contains initialized data, as well as uninitialized data like global variables.
2. **Code Segment**: Holds the program's executable instructions.
3. **Shared Library Segment**: Specifies which shared libraries should be loaded during execution.

![image.png](image.png)

# PROGRAM header

oh shit away go again an other header struct ! i swear that the last one !

the program header describes how sections of the file should be mapped into memory for execution. It is used by the operating system's loader to determine which parts of the ELF file to load, where to load them in memory, and how to set their permissions (read, write, execute). Each entry in the program header corresponds to a segment, such as code, data, or dynamic linking information.

```c
typedef struct {
    uint32_t p_type;   // Segment type
    uint32_t p_flags;  // Segment flags
    uint64_t p_offset; // Offset of the segment in the file
    uint64_t p_vaddr;  // Virtual address of the segment in memory
    uint64_t p_paddr;  // Physical address (not relevant on most platforms, use same as p_vaddr)
    uint64_t p_filesz; // Size of the segment in the file
    uint64_t p_memsz;  // Size of the segment in memory
    uint64_t p_align;  // Alignment of the segment
} Elf64_Phdr;
```

### `p_type`

```c
typedef enum {
    PT_NULL    = 0,       // Unused segment
    PT_LOAD    = 1,       // Loadable segment
    PT_DYNAMIC = 2,       // Dynamic linking information
    PT_INTERP  = 3,       // Interpreter path
    PT_NOTE    = 4,       // Auxiliary information (for tool lke debuger)
    PT_SHLIB   = 5,       // Reserved
    PT_PHDR    = 6,       // Program header itself
    PT_TLS     = 7,       // Thread-Local Storage
    PT_LOOS    = 0x60000000,  // OS-specific
    PT_HIOS    = 0x6fffffff,  // OS-specific
    PT_LOPROC  = 0x70000000,  // Processor-specific
    PT_HIPROC  = 0x7fffffff   // Processor-specific
} Elf64_PType;
```

### `p_flags`

```c
typedef enum {
    PF_X = 0x1,  // Execute permission
    PF_W = 0x2,  // Write permission
    PF_R = 0x4   // Read permission
} Elf64_PFlags;
```

### `p_offset`

The `p_offset` field specifies the offset in the file where the segment starts if it has any.

### `p_vaddr`

The virtual address at which the segment is loaded into memory.

### `p_paddr`

The physical address of the segment, typically not used in most systems but can be important in embedded systems. For most systems, this is the same as `p_vaddr`.

### `p_filesz`

The size in bytes of the segment in the file (i.e., how much data to load from the file). if its zero the segment will only be define by the header

### `p_memsz`

The size in memory of the segment, which may be larger than `p_filesz` if the segment contains uninitialized data (e.g., `.bss` section).

### `p_align`

The alignment requirements of the segment in memory and in the file. For example, loadable segments are often aligned to page size (4096 bytes).

## help

you can see program header with the command

```c
readelf --segments real_program
```

i lie…. THERE IS AN OTHER HEADER THE :

# SECTION HEADER

The **section header table** is primarily used by the linker and debugger to describe the sections of the file (e.g., `.text`, `.data`, symbol tables). Each section has information about its type, size, and location in the file.

```c
typedef struct {
    uint32_t sh_name;      // Section name (offset into the section header string table)
    uint32_t sh_type;      // Section type (SHT_PROGBITS, SHT_SYMTAB, etc.)
    uint64_t sh_flags;     // Section attributes (writable, executable, etc.)
    uint64_t sh_addr;      // Virtual address in memory where the section will be loaded
    uint64_t sh_offset;    // Offset of the section in the file
    uint64_t sh_size;      // Size of the section in bytes
    uint32_t sh_link;      // Link to another section (depends on type)
    uint32_t sh_info;      // Additional information (depends on type)
    uint64_t sh_addralign; // Alignment of the section
    uint64_t sh_entsize;   // Size of entries, if the section holds a table
} Elf64_Shdr;

```

### `sh_type`

This field specifies the type of the section, and its value can be one of the following:

```c
typedef enum {
    SHT_NULL     = 0,      // Section header is inactive
    SHT_PROGBITS = 1,      // Program information
    SHT_SYMTAB   = 2,      // Symbol table
    SHT_STRTAB   = 3,      // String table
    SHT_RELA     = 4,      // Relocation entries with addends
    SHT_HASH     = 5,      // Symbol hash table
    SHT_DYNAMIC  = 6,      // Dynamic linking information
    SHT_NOTE     = 7,      // Auxiliary information
    SHT_NOBITS   = 8,      // Uninitialized data (e.g., .bss)
    SHT_REL      = 9,      // Relocation entries without addends
    SHT_DYNSYM   = 11      // Dynamic symbol table
} Elf64_ShType;
```

---

### `sh_flags`

The `sh_flags` field contains attributes of the section, such as whether it is writable, executable, or allocatable in memory:

```c
typedef enum {
    SHF_WRITE       = 0x1,    // Section contains writable data
    SHF_ALLOC       = 0x2,    // Section is allocated in memory
    SHF_EXECINSTR   = 0x4     // Section contains executable instructions
} Elf64_ShFlags;
```

# Key ELF Sections

ELF files are composed of various sections, each with a specific purpose. Here are some of the most important sections:

- **`.text`**: Contains the program's executable code (instructions).
- **`.data`**: Contains initialized global and static variables.
- **`.bss`**: Contains uninitialized global and static variables.
- **`.rodata`**: Contains read-only data, such as string literals.
- **`.symtab`**: A symbol table, used during linking, that contains information about functions and variables.
- **`.strtab`**: A string table containing the names of symbols referenced in the symbol table.
- **`.rel` and `.rela`**: Sections that contain relocation information.

# Symbol Tables and Relocations

### Symbol Table

- **`.symtab`** (Symbol Table): This section contains the names and attributes of symbols (e.g., functions, variables) in the program. It is used by the linker to resolve addresses and is referenced by the string table.
- **`.dynsym`** (Dynamic Symbol Table): A smaller version of the symbol table used for dynamic linking.

```c
typedef struct {
    uint32_t st_name;   // Offset into the string table for the symbol name
    uint8_t  st_info;   // Type and binding attributes of the symbol
    uint8_t  st_other;  // Visibility of the symbol
    uint16_t st_shndx;  // Section index the symbol is in
    uint64_t st_value;  // Symbol value (e.g., address)
    uint64_t st_size;   // Size of the symbol
} Elf64_Sym;
```

### Relocation Sections

- **Relocation Entries** are used to adjust symbol references when linking or loading. Sections like `.rel` and `.rela` contain relocation information that helps the linker and loader patch addresses.

```c
typedef struct {
    uint64_t r_offset;   // Offset where relocation applies
    uint64_t r_info;     // Symbol and type of relocation
    int64_t  r_addend;   // Constant addend (only in RELA relocations)
} Elf64_Rela;
```

# ELF and Dynamic Linking

ELF supports dynamic linking through shared libraries. The **dynamic segment** (`PT_DYNAMIC`) contains information for the runtime linker to resolve symbols and load shared libraries.

- **`.dynsym`**: The dynamic symbol table, which contains symbols that need to be dynamically resolved.
- **`.dynstr`**: A string table for the names of symbols in `.dynsym`.
- **`.plt` and `.got`**: The Procedure Linkage Table (PLT) and Global Offset Table (GOT) help in the process of lazy loading and resolving function addresses at runtime.

# EXAMPLE

let’s start with a small example

```c
#include <stdio.h>

void print_hello_world()
{
	printf("hello world\n");
}

int main()
{
	print_hello_world();
	print_hello_world();
	print_hello_world();
}
```

---

that we will compile and and analyse with the command `hexdump -C program_name` or the `readelf`  command for prettier formatting but let’s start with a raw hexdump for the e_ident part of the elf header

```bash
> hexdump -C test | head -1
00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
```

with the first 4 bytes `7f 45 4c 46` we now that this is a ELF  

with the next byte `02` that we are on x86-64

with the next byte `01`  that we use little endian formatting

with the next byte `01`  Oh My Goad we are on elf version one !

with the next 2 byte `00  00` we see that we get no ABI

raw output is fun but lets have a ✨ prettier output ✨

```bash
**> readelf -h test
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1060
  Start of program headers:          64 (bytes into file)
  Start of section headers:          14016 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         31
  Section header string table index: 30**

```

that a lot of information but what we really want to remember is that there is `13` program header of `56` bytes each

---

![image.png](image%201.png)

we get the program header that we can analyse with the `readelf —segments` 

```bash
readelf --segments test

Elf file type is DYN (Position-Independent Executable file)
Entry point 0x1060
There are 13 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000002d8 0x00000000000002d8  R      0x8
  INTERP         0x0000000000000318 0x0000000000000318 0x0000000000000318
                 0x000000000000001c 0x000000000000001c  R      0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000628 0x0000000000000628  R      0x1000
  LOAD           0x0000000000001000 0x0000000000001000 0x0000000000001000
                 0x000000000000019d 0x000000000000019d  R E    0x1000
  LOAD           0x0000000000002000 0x0000000000002000 0x0000000000002000
                 0x000000000000011c 0x000000000000011c  R      0x1000
  LOAD           0x0000000000002db8 0x0000000000003db8 0x0000000000003db8
                 0x0000000000000258 0x0000000000000260  RW     0x1000
  DYNAMIC        0x0000000000002dc8 0x0000000000003dc8 0x0000000000003dc8
                 0x00000000000001f0 0x00000000000001f0  RW     0x8
  NOTE           0x0000000000000338 0x0000000000000338 0x0000000000000338
                 0x0000000000000030 0x0000000000000030  R      0x8
  NOTE           0x0000000000000368 0x0000000000000368 0x0000000000000368
                 0x0000000000000044 0x0000000000000044  R      0x4
  GNU_PROPERTY   0x0000000000000338 0x0000000000000338 0x0000000000000338
                 0x0000000000000030 0x0000000000000030  R      0x8
  GNU_EH_FRAME   0x0000000000002010 0x0000000000002010 0x0000000000002010
                 0x000000000000003c 0x000000000000003c  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000002db8 0x0000000000003db8 0x0000000000003db8
                 0x0000000000000248 0x0000000000000248  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.gnu.property .note.gnu.build-id .note.ABI-tag .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt 
   03     .init .plt .plt.got .plt.sec .text .fini 
   04     .rodata .eh_frame_hdr .eh_frame 
   05     .init_array .fini_array .dynamic .got .data .bss 
   06     .dynamic 
   07     .note.gnu.property 
   08     .note.gnu.build-id .note.ABI-tag 
   09     .note.gnu.property 
   10     .eh_frame_hdr 
   11     
   12     .init_array .fini_array .dynamic .got 

```

## PHDR (Program Header Table)

```bash
Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000002d8 0x00000000000002d8  R      0x8
```

the PHDR is the section that we are currently looking.

We can recheck that this start at 0x40 (64) bytes like we said earlier and is 0x2d8 (728) bytes like the number of header `13` and the sizeof each header `56` `13 * 56 == 728` 

## **INTERP** (Interpreter Path)

```bash
Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
INTERP         0x0000000000000318 0x0000000000000318 0x0000000000000318
                 0x000000000000001c 0x000000000000001c  R      0x1
```

we see that at offset 0x318 we have a string of 0x1c bytes that we need to interpret as a path 

```bash
00000310  01 00 00 00 00 00 00 00  2f 6c 69 62 36 34 2f 6c  |......../lib64/l|
00000320  64 2d 6c 69 6e 75 78 2d  78 38 36 2d 36 34 2e 73  |d-linux-x86-64.s|
```

the /lib64/ld-linux-x86-64.so is the dynamic linker file that will link all linux dependency like the libc where we could find the printf() function we use in the test code 

## LOAD

```bash
Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000628 0x0000000000000628  R      0x1000
  LOAD           0x0000000000001000 0x0000000000001000 0x0000000000001000
                 0x000000000000019d 0x000000000000019d  R E    0x1000
  LOAD           0x0000000000002000 0x0000000000002000 0x0000000000002000
                 0x000000000000011c 0x000000000000011c  R      0x1000
  LOAD           0x0000000000002db8 0x0000000000003db8 0x0000000000003db8
                 0x0000000000000258 0x0000000000000260  RW     0x1000
```

**LOAD** segment in the ELF **program headers** is used to specify which parts of the ELF file should be loaded into memory and with what permissions. Each **LOAD** segment describes a portion of the file that will be mapped into memory, and it provides details about access permissions like **read**, **write**, or **execute**.

## **DYNAMIC** (Dynamic Linking Information)

```bash
Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
DYNAMIC        0x0000000000002dc8 0x0000000000003dc8 0x0000000000003dc8
                 0x00000000000001f0 0x00000000000001f0  RW     0x8
```

The **DYNAMIC** segment contains dynamic linking information. This segment is used by the **dynamic linker/loader** (in this case, `/lib64/ld-linux-x86-64.so.2`) to resolve symbols and load shared libraries at runtime.

## **NOTE** (Auxiliary Information)

```bash
Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
NOTE           0x0000000000000338 0x0000000000000338 0x0000000000000338
                 0x0000000000000030 0x0000000000000030  R      0x8
```

The **NOTE** segment contains auxiliary information that is typically used for things like build IDs, OS-specific metadata, and process annotations.

## **GNU_PROPERTY** (GNU Property Notes)

```bash
Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
GNU_PROPERTY   0x0000000000000338 0x0000000000000338 0x0000000000000338
                 0x0000000000000030 0x0000000000000030  R      0x8
```

The **GNU_PROPERTY** segment stores additional metadata specific to the GNU toolchain. This metadata can include details about processor features or security properties (such as stack protection or control flow integrity).

## **GNU_EH_FRAME** (Exception Handling Frame)

```bash
Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
GNU_EH_FRAME   0x0000000000002010 0x0000000000002010 0x0000000000002010
                 0x000000000000003c 0x000000000000003c  R      0x4
```

The **GNU_EH_FRAME** segment contains exception-handling information (for C++ or other languages that support exceptions). It is used to unwind the stack when an exception is thrown.

### **GNU_STACK** (Stack Permissions)

```
Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
```

The **GNU_STACK** segment specifies the stack's memory permissions. In this case, the stack is **readable** and **writable** (`RW`) but not executable (this is a security feature to prevent execution of code on the stack, known as "stack smashing protection").

## **GNU_RELRO** (Read-Only After Relocation)

```bash
Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
GNU_RELRO      0x0000000000002db8 0x0000000000003db8 0x0000000000003db8
                 0x0000000000000248 0x0000000000000248  R      0x1
```

The **GNU_RELRO** segment stands for "Read-Only After Relocation." This segment is initially **writable** during program startup while relocations are being applied, and then it is made **read-only** to protect against certain types of security attacks (e.g., tampering with function pointers).

# SECTION HEADER

let’s take the elf header to now how many of witch size and at witch offset we have to to analyse

```bash
> readelf -h test | grep section
Start of section headers:          14016 (bytes into file)
Size of section headers:           64 (bytes)
Number of section headers:         31
```

we can see then stylise with the `readelf -S`

```bash
Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000000318  00000318
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.gnu.pr[...] NOTE             0000000000000338  00000338
       0000000000000030  0000000000000000   A       0     0     8
  [ 3] .note.gnu.bu[...] NOTE             0000000000000368  00000368
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .note.ABI-tag     NOTE             000000000000038c  0000038c
       0000000000000020  0000000000000000   A       0     0     4
  [ 5] .gnu.hash         GNU_HASH         00000000000003b0  000003b0
       0000000000000024  0000000000000000   A       6     0     8
  [ 6] .dynsym           DYNSYM           00000000000003d8  000003d8
       00000000000000a8  0000000000000018   A       7     1     8
  [ 7] .dynstr           STRTAB           0000000000000480  00000480
       000000000000008d  0000000000000000   A       0     0     1
  [ 8] .gnu.version      VERSYM           000000000000050e  0000050e
       000000000000000e  0000000000000002   A       6     0     2
  [ 9] .gnu.version_r    VERNEED          0000000000000520  00000520
       0000000000000030  0000000000000000   A       7     1     8
  [10] .rela.dyn         RELA             0000000000000550  00000550
       00000000000000c0  0000000000000018   A       6     0     8
  [11] .rela.plt         RELA             0000000000000610  00000610
       0000000000000018  0000000000000018  AI       6    24     8
  [12] .init             PROGBITS         0000000000001000  00001000
       000000000000001b  0000000000000000  AX       0     0     4
  [13] .plt              PROGBITS         0000000000001020  00001020
       0000000000000020  0000000000000010  AX       0     0     16
  [14] .plt.got          PROGBITS         0000000000001040  00001040
       0000000000000010  0000000000000010  AX       0     0     16
  [15] .plt.sec          PROGBITS         0000000000001050  00001050
       0000000000000010  0000000000000010  AX       0     0     16
  [16] .text             PROGBITS         0000000000001060  00001060
       0000000000000130  0000000000000000  AX       0     0     16
  [17] .fini             PROGBITS         0000000000001190  00001190
       000000000000000d  0000000000000000  AX       0     0     4
  [18] .rodata           PROGBITS         0000000000002000  00002000
       0000000000000010  0000000000000000   A       0     0     4
  [19] .eh_frame_hdr     PROGBITS         0000000000002010  00002010
       000000000000003c  0000000000000000   A       0     0     4
  [20] .eh_frame         PROGBITS         0000000000002050  00002050
       00000000000000cc  0000000000000000   A       0     0     8
  [21] .init_array       INIT_ARRAY       0000000000003db8  00002db8
       0000000000000008  0000000000000008  WA       0     0     8
  [22] .fini_array       FINI_ARRAY       0000000000003dc0  00002dc0
       0000000000000008  0000000000000008  WA       0     0     8
  [23] .dynamic          DYNAMIC          0000000000003dc8  00002dc8
       00000000000001f0  0000000000000010  WA       7     0     8
  [24] .got              PROGBITS         0000000000003fb8  00002fb8
       0000000000000048  0000000000000008  WA       0     0     8
  [25] .data             PROGBITS         0000000000004000  00003000
       0000000000000010  0000000000000000  WA       0     0     8
  [26] .bss              NOBITS           0000000000004010  00003010
       0000000000000008  0000000000000000  WA       0     0     1
  [27] .comment          PROGBITS         0000000000000000  00003010
       000000000000002b  0000000000000001  MS       0     0     1
  [28] .symtab           SYMTAB           0000000000000000  00003040
       0000000000000378  0000000000000018          29    18     8
  [29] .strtab           STRTAB           0000000000000000  000033b8
       00000000000001ec  0000000000000000           0     0     1
  [30] .shstrtab         STRTAB           0000000000000000  000035a4
       000000000000011a  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  D (mbind), l (large), p (processor specific)

```

the scheme repeate we can parse with the struct each Section header and get the data

### reminder

- **PROGBITS**: This is the most common type, used for program code, data, and various types of metadata.
- **NOBITS**: Indicates that the section does not take up space in the file but will be allocated memory when the program is loaded (like `.bss`).
- **SYMTAB**: The symbol table, which holds information about functions, variables, and their locations.
- **STRTAB**: The string table, which holds the names of symbols and sections.