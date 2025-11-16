#clang #llvm 
# **Backend Link**

We’ve parsed a complete top-level statement and now need to forward it to the backend for code generation. This section clarifies how Clang transitions from parsing to emitting IR via `CodeGenModule::EmitTopLevelDecl`:

## Parsing Loop (`clang::ParseAST`)

```cpp
for (bool AtEOF = P.ParseFirstTopLevelDecl(ADecl, ImportState);
     !AtEOF;
     AtEOF = P.ParseTopLevelDecl(ADecl, ImportState)) {
  if (ADecl && !Consumer->HandleTopLevelDecl(ADecl.get()))
    return;
}
```

* **`ParseFirstTopLevelDecl` / `ParseTopLevelDecl`**: Retrieve the next `DeclGroupRef` from the parser
* **`Consumer->HandleTopLevelDecl`**: Invokes the backend-specific handler with the parsed declaration

## Backend Dispatch Chain

1. **`BackendConsumer::HandleTopLevelDecl`**

   * Entry point from the frontend to the code-generation backend
2. **`CodeGeneratorImpl::HandleTopLevelDecl`**

   * Iterates over each `Decl` in the `DeclGroupRef`, passing them on to the builder
3. **`CodeGen::CodeGenModule::EmitTopLevelDecl`**

   * Final step to inspect and emit each individual `Decl`


## **CodeGenModule::EmitTopLevelDecl**

* **Dispatch Role**: Determines the nature of the `Decl` and delegates accordingly with a switch case
* **`EmitGlobal(GlobalDecl)`**: Converts `FunctionDecl`/`VarDecl` into LLVM IR objects
* **Contextual Processing**: For containers like namespaces or records, it may recurse

## **CodeGenModule::EmitGlobal**

* If weak attribute don't emit code by themselves
* If alias return `EmitAliasDefinition`
* If ifunc return `EmitIFuncDefinition`
* If cpu_dispatch return `emitCPUDispatchDefinition`
* Handle cuda
* Handle OpenMP
* If function decl
    * handle if attribute `annotate("...")`
    * if func dosent have a body emit the code (`GetOrCreateLLVMFunction`)
* Else variable decl
    * if openMP
    * Ensure strong inline variable are emitted
* Emit required global immediately if possible (`EmitGlobalDefinition`)
* Track C++ global initializer order
* Emit used declaration later
* Defer required but non-eager decl
* Register lazy deferred declaration

## **CodeGenModule::EmitGlobalDefinition**

* If function
    * Handle c++ specific
    * If multi version (`target_clones("avx2","sse2"`) `EmitMultiVersionFunctionDefinition`
    * Else `EmitGlobalFunctionDefinition`
* Else `EmitGlobalVarDefinition`

## **CodeGenModule::EmitGlobalFunctionDefinition**

* Get function info
* Get function type
* Get or create the func proto
* If already emit return
* Chooses `external` vs internal `linkage` based on `static`
* Adds attributes (`nounwind`, `uwtable`, calling-conv, etc.)
* Ensures C-name mangling but still internal linkage
* Added `comdat` + `weak_odr` on the inline function 
* `GenerateCode`
* Set attribute
* If a `Constructor` or `Destructor` add it to the right table
* If OpenMP add emit target function

## **CodeGenFunction::GenerateCode**

* Build function arg list
* Handle if the function is a `inline`
* Replace inline stub with definition
* If no debug clear all info
* Distinguish a function definition from a proto definition
* Find the location
* Handle function from template
* Handle coroutines
* Start function
* Switch case for multiple special case 
    * C++ `constructor`/`destructor`/`method`...
    * Else `EmitFunctionBody`
* Inserting (as configured) a sanitizer check or trap and an unreachable before the closing brace
* Finish function
* Check if you can make the function a `not throw`
    
## **CodeGenFunction::EmitFunctionBody**
To be explained
