# Symbol and string tables
- `DT_SYMTAB` → `ElfW(Sym)*` symbol table    
- `DT_SYMENT` → sizeof(ElfW(Sym)), sanity check
- `DT_STRTAB` → `char*` string table
- `DT_STRSZ` → size of string table (optional sanity / bounds)
- `DT_HASH` → old SysV hash table (access the same data then symtab)
- `DT_GNU_HASH` → GNU hash table (access the same data then symtab)

## Aditional part
- TLS
- constructor
- destructor

# Access

## REL
addend inside the section bytes (base + symbol_value)
- `DT_REL` -> relocation table address
- `DT_RELSZ` -> total size
- `DT_RELENT` -> entry size
## RELA 
addend inside relocation table (base + symbol_value + addent)
- `DT_RELA` → relocation table address
- `DT_RELASZ` → total size
- `DT_RELAENT`→ entry size
## JMPREL
plt entry
- `DT_JMPREL` → table for PLT relocations
- `DT_PLTREL` → type: `DT_REL` or `DT_RELA` (aarch64 and x86-64 use rela)
- `DT_PLTRELSZ` → size of the PLT relocation table
- `DT_PLTGOT` → GOT used by PLT code (Arch-specific usage)
### PLT resolution mode 
you can resolve every plt symbol at the start or put stub to resolve them at first use

## OTHER
- `DT_NEEDED`                     (if you also handle dependencies)
- `DT_VERSYM` / `DT_VERNEED`*       (if you want correct versioning later)

# order of resolution
- Phase 1: only REL/RELA + basic PLT + NEEDED
- Phase 2: symbol versioning
- Phase 3: TLS, RELRO, IFUNC, etc.
