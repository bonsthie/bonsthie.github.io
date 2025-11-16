#llvm
# compilation

## cmake flags

* `-L`: list all variable
* `-H`: help
* `-A`: all (that a lot normaly not needed)


* `-D<var>=<value>`: Sets a CMake variable to the specified value.
* `-G<generatorName>`: Specifies the build system generator to use.
* `-C<pathToCacheFile>`: Preloads the CMake cache with values from the given file.

```sh
-DLLVM_ENABLE_PROJECT=<project1>;<project2>
-DLLVM_TARGESTS_TO_BUILD=<Target1>;<Target2>
-DCMAKE_BUILD_TYPE= # Debug or Release

-DLLVM_OPTIMIZED_TABLEGEN=1
-DBUILD_SHARED_LIBS=1
-DBUILD_SHARED_LIBS=1
-DLLVM_ENABLE_ASSERTIONS=1

-GNinja
```

## ninja

```sh
ninja             # Build all
ninja <target>    # Build specific target
ninja clean       # Clean build
ninja check       # Run tests
ninja intrinsics_gen #gen the TableGen file
```

### Useful Flags

```sh
-j <n>            # Parallel jobs (use for -j1 with -v to see what happen)
-v                # Verbose output
-d explain        # Debug why a rule runs
-t targets all    # List targets
-t compdb cxx     # Export compile_commands.json
```

