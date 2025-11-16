# **Driver-Level Setup**
#clang #llvm 
```
clang_main
 ├─> parse driver arguments & initialize Driver
 ├─> if -cc1 execute the compiler
 ├─> BuildCompilation
 │    ├─> TranslateInputArgs (raw args → TranslatedArgs)
 │    ├─> Configure ToolChains (host & offload)
 │    ├─> Validate flags & compute TargetTriple
 │    ├─> Determine driver-mode (gcc/g++/cpp/clang-cl)
 │    ├─> Plan phases (preprocess → compile → assemble → link)
 │    ├─> create Action DAG (nodes for each compilation step)
 │    └─> BuildJobs
 │         └─> create Job list (Command objects)
 └─> ExecuteCompilation
      ├─> fork & exec each Job in DAG order (parallel where possible)
      └─> collect exit codes & handle failures

```

## **clang\_main**

* **set clang_main has the bottom of the stack**
* **Setup bug report message**
* On Windows, reopen any missing `stdin`/`stdout`/`stderr` handles to `NUL`
* **Initialize all architecture targets** (x86, ARM, AArch64, etc.)
* **Create** a `BumpPtrAllocator` and `StringSaver` to own strings from `@file` response files
* **Get** the program name (e.g. `bin/clang`)
* **Determine** if arguments are MSVC-style (`clang-cl` mode)
* **`expandResponseFiles`**: splice tokens from any `@file` entries into the argument list
* **If** this is a `-cc1` invocation (seen later via `ExecuteCC1Tool`)
* **Parse early settings** like `-canonical-prefixes` / `-no-canonical-prefixes`
* **Handle** CL-mode environment variables (`CL`, `_CL_`) for MSVC overrides
* **Apply** `CCC_OVERRIDE_OPTIONS` overrides (e.g. `CCC_OVERRIDE_OPTIONS="# O0 +-g"`)
  * `#` silences the “### Removing …” / “### Adding …” messages
  * Replaces any `-O*` flags with `-O0`
  * Appends `-g` to the flag list
* **GetExecutablePath**: compute the absolute or raw driver path
* **Parse** `-fintegrated-cc1` / `-fno-integrated-cc1` to choose in-process vs. external cc1
* **Setup** the diagnostic engine
  * **Parse** DiagOpts (`-Werror`, `-pedantic`, etc.)
  * **Instantiate** `TextDiagnosticPrinter` and `DiagnosticsEngine`
* **Initialize** the filesystem VFS and process warning options
* **Initialize** `TheDriver` (GCC-like driver main body)
* **Configure** extra arguments (target, mode, prefix)
* **Assign** `TheDriver.CC1Main = ExecuteCC1WithContext` if using in-process cc1
* **Build** the `Compilation` via `BuildCompilation` (preprocess, compile, assemble, link)
* **Handle** errors/crashes during `Compilation` creation (Windows specifics, reproducers)
* **Execute** compilation jobs: `TheDriver.ExecuteCompilation(*C, FailingCommands)`
* **Handle** compilation errors/crashes and emit diagnostics

---

## **BuildCompilation**

* **Get driver mode** (`--driver-mode=g++`)

  Default mapping:

  ```bash
  clang foo.c      # DriverMode == "gcc"
  clang++ foo.cpp  # DriverMode == "g++"
  clang -E foo.c   # DriverMode == "cpp"
  clang-cl foo.c   # DriverMode == "cl"
  ```

* **Parse** command-line arguments into `CLOptions`

* **Load** config file (`~/.config/clang/config`) via `loadConfigFiles`

  **Order of override**:

  1. Head defaults (`CfgOptionsHead`)
  2. User arguments (`CLOptions`)
  3. Tail overrides (`CfgOptionsTail`)

* **Fuse** head + user + tail into a single argument set
* **Translate** input args via `TranslateInputArgs(*UArgs)` → `TranslatedArgs`
* **Claim** flags to suppress unused warnings:
  * `-canonical-prefixes` / `-no-canonical-prefixes`
  * `-fintegrated-cc1` / `-fno-integrated-cc1`
* **Handle** hidden debug flags (`-ccc-print-phases`, `-ccc-print-bindings`)
* **Setup** MSVC or DXC/HLSL→Vulkan/SPIR-V modes
* **Configure** target and install directories (`COMPILER_PATH`, etc.)
* **Compute** `TargetTriple` from `--target`, `-m*`, driver-mode
* **Select** toolchain via `getToolChain`
* **Validate/warn** on triple vs. object-format mismatches
* **Append** multilib macros from `TC.getMultilibMacroDefinesStr`
* **Invoke** phases: preprocess → compile → assemble → link
* **Apply** architecture-specific settings (\~50 lines)
* **Initialize** the `Compilation` class
* **Call** `BuildJobs(*C)` to schedule compile processes

---

## **Compilation** class

``` cpp
Compilation::Compilation(const Driver &D,
                         const ToolChain &_DefaultToolchain,
                         InputArgList *_Args,
                         DerivedArgList *_TranslatedArgs,
                         bool ContainsError)
    : TheDriver(D),
      DefaultToolchain(_DefaultToolchain),
      Args(_Args),
      TranslatedArgs(_TranslatedArgs),
      ContainsError(ContainsError) {
  // The offloading host toolchain is the default toolchain.
  OrderedOffloadingToolchains.insert(
      std::make_pair(Action::OFK_Host, &DefaultToolchain));
}
```

* **TheDriver**: Reference to the `Driver` controlling this compilation.
* **DefaultToolchain**: Host `ToolChain` for codegen.
* **Args** / **TranslatedArgs**: Raw and translated argument lists.
* **ContainsError**: Error flag from initial parsing.
* **OrderedOffloadingToolchains**: Inserts **`Action::OFK_Host`** → host toolchain.

---

## **BuildJobs**

### **Action**:
An abstract compilation step (a node in the DAG (Directed Acyclic Graph)). you can see them with `-ccc-print-phases`

**Example:**

```bash
 ➜ clang -ccc-print-phases foo.c bar.c

            +- 0: input, "foo.c", c
         +- 1: preprocessor, {0}, cpp-output
      +- 2: compiler, {1}, ir
   +- 3: backend, {2}, assembler
+- 4: assembler, {3}, object
|           +- 5: input, "bar.c", c
|        +- 6: preprocessor, {5}, cpp-output
|     +- 7: compiler, {6}, ir
|  +- 8: backend, {7}, assembler
|- 9: assembler, {8}, object
10: linker, {4, 9}, image
 ```

### **Job**
A Job is the concrete command-line execution that performs an Action defined in Clang’s compilation DAG. you can see them with the `-ccc-print-bindings` or in detail with `-###`
```bash
 ➜ clang -ccc-print-bindings foo.c bar.c

# "x86_64-unknown-linux-gnu" - "clang", inputs: ["foo.c"], output: "/tmp/foo-4040da.o"
# "x86_64-unknown-linux-gnu" - "clang", inputs: ["bar.c"], output: "/tmp/bar-15d3e6.o"
# "x86_64-unknown-linux-gnu" - "GNU::Linker", inputs: ["/tmp/foo-4040da.o", "/tmp/bar-15d3e6.o"], output: "a.out"
```

**Driver::BuildJobs steps:**

* **Count outputs** (including `.ifs/.ifso` exceptions for `-o`)
* **Handle** Mach-O multi-arch (`-arch`) specifics
* Collect the list of architectures
* **For each Action**:
  * Determine `LinkingOutput` for `LipoJobAction`
  * Call `BuildJobsForAction` to create Jobs
* **Disable integrated-cc1** if `C.getJobs().size() > 1`
* **Print stats** if `CCPrintProcessStats` is enabled
* looking for an `assembler` job to set a `HasAssembleJob` bool
* **Return** based on errors or `-Qunused-arguments`
* **Suppress** warnings for flags like `-fdriver-only`, `-###`, etc.
* **Warn** on any remaining unclaimed or unsupported arguments

## **ExecuteCompilation**
> this is call in clang_main

* **LogOnly mode**: if `-fdriver-only`, print jobs (`-v`) then execute in *log-only* mode.
* **Dry run**: if `-###`, print jobs and exit based on diagnostic errors.
* **Error check**: abort early if any diagnostic errors before execution.
* **Response files**: set up response files for each job when needed.
* **Execute jobs**: run `C.ExecuteJobs`, collecting failures in `FailingCommands`.
* **Fast exit**: return `0` when there are no failures.
* **Cleanup**: on failure and when not saving temps, remove produced result files; for crashes (`<0`), also clean failure files.
* **Signal handling**: ignore `EX_IOERR` (SIGPIPE) without printing diagnostics.
* **Detailed diagnostics**: if a failing tool lacks good diagnostics or exit code ≠1, emit `err_drv_command_failed` or `err_drv_command_signalled`.
* **Return code**: propagate the first non-zero or special exit code in `Res`.

##  **ExecuteJobs**

In Unix, all jobs are executed regardless of whether an error occurs
In MSVC’s `CLMode`, jobs stop when an error occurs

on all the jobs :
 * call the function ExecuteCommand
 * if fail store in the failing command vector
    * if CLMode return 

##   **ExecuteCommand**

* print if `CC_PRINT_OPTIONS`
* Execute `C.Execute`
* manage error

##   **Execute**

* **Print file names** for logging and diagnostics.
* **Construct `Argv`**:
  * If no response file: push `Executable`, optional `PrependArg`, then `Arguments`, terminate with `nullptr`.
  * If a response file is needed:
    1. **Serialize** arguments into `RespContents` via `writeResponseFile`.
    2. **Build** `Argv` for the response file (`@file` syntax).
    3. **Write** the response file with proper encoding; on error, set `ErrMsg`/`ExecutionFailed` and return `-1`.
* **Prepare environment** (`Env`) if any variables are set.
* **Convert** `Argv` array to `StringRef` array (`Args`).
* **Handle redirects**:
  * If `RedirectFiles` are present, convert to `std::optional<StringRef>` list and call `ExecuteAndWait` with those.
  * Otherwise, call `ExecuteAndWait` with the provided `Redirects`.
* **Execute and wait**:
  * `llvm::sys::ExecuteAndWait` forks, execs `Executable` with `Args` and `Env`, applies redirects, collects exit code in `ErrMsg`/`ExecutionFailed`, and records `ProcStat`.
* **Return** the child process exit code (or `-1` on exec failure).
