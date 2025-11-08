#thread #debuging

## Using `top` to Monitor Threads

When debugging multi-threaded programs, `top` can help monitor individual threads within a process. You can launch your program in the background and immediately attach `top` to it:

```sh
./program & top -H -p $!
```

### Explanation:

- `./program &` → Runs the program in the background.
- `$!` → Captures the PID of the last background process.
- `top -H -p $!` → Shows the threads (`-H` flag) of the process with the given PID.

## Inspecting Memory Mappings with `/proc/$PID/maps`

To analyze memory allocations of a running process, use `/proc/$PID/maps`:

```sh
./program & export PID=$! && cat /proc/$PID/maps
```

### Explanation:

- `./program &` → Runs the program in the background.
- `export PID=$!` → Stores the PID for later use.
- `cat /proc/$PID/maps` → Displays the memory layout of the process.

### Understanding `/proc/$PID/maps`

This file contains memory regions mapped into the process’s address space. Example output:

```
00400000-00452000 r-xp 00000000 fd:01 123456 /home/user/program
00651000-00652000 r--p 00051000 fd:01 123456 /home/user/program
00652000-00653000 rw-p 00052000 fd:01 123456 /home/user/program
7f8c3b000000-7f8c3b021000 rw-p 00000000 00:00 0 
7f8c3b021000-7f8c3b800000 ---p 00000000 00:00 0 
```

### Key Fields:

- **Address range** (e.g., `00400000-00452000`) → Start and end addresses.
- **Permissions**:
    - `r` → Read
    - `w` → Write
    - `x` → Execute
    - `p` → Private mapping
- **Offset** (e.g., `00000000`) → File offset for mapped files.
- **Device & inode** (e.g., `fd:01 123456`) → Mapped file details.
- **File mapping** (e.g., `/home/user/program`) → The associated file if applicable.

## Practical Use Cases

- **Tracking high CPU usage in threads** → `top -H -p $PID`
- **Inspecting memory layout for debugging segmentation faults** → `/proc/$PID/maps`
- **Checking shared libraries and stack mappings** → Look for mapped `.so` files and `[stack]`

This is useful when debugging memory-related issues and analyzing thread behavior in multi-threaded applications.

### **`ps` to Inspect Threads and CPU Usage**

```sh
./program & ps -T -p $!
```

- `-T` → Show threads of the given PID.
- `-p` → Attach to a specific process.

### **`perf` for CPU Profiling**

```sh
./program & export PID=$! && perf record -g -p $PID perf report
```

- `-g` → Capture call graphs.
- `-p` → Attach to a process.

### **`strace` to Trace System Calls of Threads**


```sh
./program & strace -f -p $!
```

- `-f` → Follow threads.
- `-p` → Attach to a process.

## `valgrind` `helgrind`  ofc 🙃