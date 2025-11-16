# **Compiler Core `cc1`**
#clang #llvm 
## **Going back to the clang\_start**

Ok, so let's resume. We:

* created the driver settings
* with that, created the compilation settings
* created actions for each part of the compilation
* created a list of jobs from the list of actions

Now it's time to execute the job!!!

At the beginning, we talked about an `if`: the first argument of the function is `-cc1`.
That's it—we are building, guys!!!

## **Main class explanation**

Before we start, I think it's best to explain all the main classes involved in the compilation:
`CompilerInstance`, `FileManager`, `SourceManager`, `PreProcessor`, `Lexer`.
This is more of a reference section to better understand the rest.

### **`CompilerInstance`**

This is the class responsible for handling the `cc1` part of Clang.
It contains all the classes needed for compilation.

Main classes inside the `CompilerInstance`:

* `FileManager`
* `SourceManager`
* `TargetInfo`
* `PreProcessor`
* `ASTContext`, `ASTConsumer`, `ASTReader`

Function:

```c
bool CompilerInstance::ExecuteAction(FrontendAction &Act);
```

### **`FileManager`**

The `FileManager` class finds/reads files on disk or `VFS` and caches metadata and open memory buffers.
This returns a `FileEntryRef`.

> VFS: is an LLVM abstraction that makes all file sources (disk files, in-memory buffers, archives) look like a single, uniform filesystem to the compiler.

Main members inside the `FileManager`:

* `FileSystem`
* `FileSystemOptions`
* `DenseMap` of real files by ID
* `DenseMap` of real directories by ID
* `SmallVector` of virtual file entries
* `SmallVector` of virtual directory entries
* `StringMap` for cached file lookups
* `StringMap` for cached directory lookups
* `OptionalFileEntryRef` for stdin

Function:

```cpp
llvm::Expected<FileEntryRef> FileManager::getFileRef(StringRef Filename,
                                                    bool OpenFile = false,
                                                    bool CacheFailure = true,
                                                    bool IsText = true);
```

### **`SourceManager`**

This class takes raw buffers from `FileManager` (`FileEntryRef`), assigns them simple integer FileIDs, and builds the data structures needed for the rest of the compiler to ask, "What line/column is offset N in FileID X?"

Main members inside the `SourceManager`:

* `SourceMgr` – contains a list of `MemoryBuffer` objects (one per `FileID`)
* `FileIDMap` – mapping from buffer index to `FileEntryRef`
* Line/column mapping tables for each buffer (e.g., `LineOffsets`)
* Macro expansion and include-location stacks
* `FileID` counter to assign unique IDs for each buffer

Functions:

```cpp
/// Create a new FileID for the given FileEntryRef and return it.
FileID SourceManager::createFileID(FileEntryRef FE,
                                   SourceLocation Loc,
                                   SrcMgr::CharacteristicKind K);

/// Retrieve character data for a given FileID and offset.
const char *SourceManager::getCharacterData(FileID FID, unsigned &Offset);
```

### **`Preprocessor`**

This class handles all pre-parsing work—macro definitions, `#include` directives, conditional compilation—and produces a stream of tokens for the parser.

Main members inside the `Preprocessor`:

* `HeaderSearch` – manages search paths and maps headers to `FileEntryRef`
* `PreprocessorOptions` – stores flags like `-D`, `-I`, macro expansion settings
* `IdentifierTable` – uniquing table for identifiers and keywords
* `Builtin::Context` – definitions for built-in macros and keywords
* `MacroInfoMap` – maps macro names to their definitions (`MacroInfo`)

Functions:

```cpp
/// Get the next token, expanding macros and handling directives.
Token Preprocessor::Lex(Token &Tok);

/// Push a new file onto the include stack, creating a fresh Lexer.
void Preprocessor::EnterSourceFile(FileID FID, bool IsMacroFile,
                                   llvm::MemoryBuffer *Buffer);
```

### **`Lexer`**

This class reads characters from a source buffer (via `SourceManager`) and produces `Token` objects for the parser.

Main members inside the `Lexer`:

* `FileID` – identifies the buffer being lexed
* `SourceManager &SM` – provides access to buffer contents and locations
* `LangOptions &LangOpts` – controls language-specific lexing behavior
* `const char *BufferStart/Ptr/End` – pointers to track current position in the memory buffer
* `Token CurToken` – storage for the current token
* `unsigned CurPPEnd` – offset for end-of-macro/file switching

Functions:

```cpp
/// Lex the next token from the buffer (skips whitespace/comments).
Token Lexer::Lex(Token &Result);

/// Initialize the lexer for a new buffer.
void Lexer::Initialize(FileID FID,
                       const LangOptions &LangOpts,
                       SourceManager &SM,
                       bool IsAtStartOfFile);
```

# **back to code**

```
cc1_main
 ├─ split & claim cc1-only flags (plugins, -mllvm, etc.)
 ├─ initialize cc1 context
 │    ├─ create DiagnosticIDs & DiagnosticsEngine + consumers
 │    ├─ register PCH formats & built-in targets
 │    └─ parse frontend/backend flags (–disable-free, –disable-llvm-verifier, –main-file-name, etc.)
 ├─ parse remaining argv into CompilerInvocation
 ├─ instantiate CompilerInstance CI
 │    ├─ attach DiagnosticsEngine
 │    ├─ set up FileManager & SourceManager
 │    ├─ set up TargetInfo/TargetMachine
 │    ├─ initialize Preprocessor & HeaderSearch
 │    └─ create ASTContext & Sema
 ├─ configure timers, timeTraceProfile & stats if requested
 ├─ status = ExecuteCompilerInvocation(CI)
 │    ├─ handle –help / –version
 │    ├─ load plugins & handle –mllvm
 │    ├─ configure sanitizers & static-analysis hooks
 │    ├─ select & construct FrontendAction Act
 │    ├─ CI.BeginSourceFile(Act, inputs)
 │    ├─ CI.ExecuteAction(Act) ← Parser→Sema→CodeGen or AST walk
 │    ├─ CI.EndSourceFile()
 │    └─ return Act success/failure
 └─ return status
```

##   **ExecuteCC1Tool**

* tokenize the cmd line 
* redirect on the right cc1

## **cc1_main**

* init a CompilerInstance() and a DiagnosticIDs() instance
* init PCH format (precompiled header)
* init all target base functions
* diag setup
* fill the `CompilerInvocation` of the `CompilerInstance` that contains all compiler settings and does some checks on the args list
* timeTraceProfile init if needed
* print clang cpu/stats
* init the diag for the `CompilerInstance`
* install the llvm backend diag in the instance
* execute `ExecuteCompilerInvocation`
* handle errors and timers

## ExecuteCompilerInvocation

* handle basic `-help` / `-version`
* load clang user plugin
* handle `-mllvm`
* optional static analyzer stuff
* error checkup
* creation of the `FrontendAction` (binds the right ExecuteAction function to the FrontendAction)
* execute the `FrontendAction`

## ExecuteAction

* Preconditions: verify diagnostics are initialized and help/version have been handled
* Diagnostics cleanup guard: ensure `getDiagnosticClient().finish()` runs on exit
* Verbose stream: obtain the verbose output stream (`OS`)
* Prepare action: invoke `Act.PrepareToExecute(*this)`, which sets up any internal state required by the action; if it returns `false`, the action cannot run and `ExecuteAction` immediately returns `false` to signal failure (rather than continuing into an invalid state)
* Create target: initialize the compilation target (architecture, vendor, OS, and ABI settings—e.g., `x86_64-unknown-linux-gnu`), configuring TargetInfo/TargetMachine; abort on failure
* ObjC rewrite patch: for Objective-C (`ObjC`) rewriting actions (e.g., `RewriteObjC`), adjust the built-in ObjCBool type (disable signed char) to match the ObjC ABI
* Verbose/stats flags: print version info if verbose, enable stats if requested
* Sort codegen tables: sort TOC (`Table of Contents`) and NoTOC variable lists (used on targets like PowerPC64 for PIC data) so lookups use binary search for efficient codegen decisions
* Process inputs: for each input file, clear IDs, run `BeginSourceFile`, `Execute`, and `EndSourceFile`
* Print diagnostic stats: emit any collected diagnostic statistics
* Dump stats to file: if `StatsFile` is set, open (or append) and write JSON stats, warning on error
* Return success: return true if no errors were reported, false otherwise

## Execute

* Get CompilerInstance
* ExecuteAction: invoke the action-specific frontend logic (`ExecuteAction()`)
* Rebuild Global Module Index: if `CI.shouldBuildGlobalModuleIndex()` and file manager/preprocessor are present, fetch `Cache = CI.getPreprocessor().getHeaderSearchInfo().getModuleCachePath()` and, if non-empty, call `GlobalModuleIndex::writeIndex`. On error, consume it silently
* Return success: always return `llvm::Error::success()` (no error propagation)

# init AST creation

## FrontendAction Hierarchy

Clang's frontend actions all inherit from the abstract base `FrontendAction`, providing a uniform `ExecuteAction` entry point. Key subclasses include:

* `ASTFrontendAction`: Runs after parsing to operate on the AST (analysis, transformations).
* `PreprocessorFrontendAction`: Hooks into the preprocessor stage (token processing before parsing).
* `CodeGenAction`: Coordinates code generation backends (e.g., IR emission, object code output).
* Other FrontendActions: Miscellaneous actions (e.g., `MergeModuleAction`, `PluginAction`).

Below is the `EmitLLVMAction`, which inherits from `CodeGenAction` and emits LLVM IR:

```cpp
class EmitLLVMAction : public CodeGenAction {
  virtual void anchor();              // Ensure vtable emission
public:
  EmitLLVMAction(llvm::LLVMContext *_VMContext = nullptr);
};

void EmitLLVMAction::anchor() { }
EmitLLVMAction::EmitLLVMAction(llvm::LLVMContext *_VMContext)
  : CodeGenAction(Backend_EmitLL, _VMContext) {}
```

What this does:

1. `anchor()`: Defines an out-of-line virtual method so that the compiler emits `EmitLLVMAction`'s vtable in this translation unit.
2. Constructor: Calls the `CodeGenAction` base with `Backend_EmitLL`, registering the action to generate LLVM IR into the given (or newly created) `LLVMContext`.

> note: all the CodeGenAction are the same, with the `llvm::LLVMContext` (`Backend_EmitLL`) being the only difference

## CodeGenAction::ExecuteAction

* Wrapper over AST path: Delegates all non-LLVM-IR inputs straight to `ASTFrontendAction::ExecuteAction()`.
* LLVM IR handling: Intercepts LLVM IR inputs and runs the IR-specific emission pipeline (bypassing the AST frontend).

> We're not detailing the LLVM IR setup and configuration steps, as those belong to the backend-specific flow.

## ASTFrontendAction::ExecuteAction

* Preprocessor check: Returns early if the preprocessor isn’t initialized.
* Stack setup: Marks the bottom of the stack to guard against deep AST recursion.
* Code completion: If requested, installs a `CodeCompleteConsumer` for IDE-style suggestions.
* Semantic analysis init: Creates/configures the `Sema` object to drive name lookup and type checking.
* Parse AST: Invokes `ParseAST`, which lexes, parses, and executes the specific frontend action logic on the AST.

# Parsing Class

### `Parser`

The `Parser` class is responsible for syntactic parsing. It takes references to the `PreProcessor` and `Sema`, and drives the parsing of tokens into declarations and statements.

### `Sema`

The `Sema` (semantic analyzer) performs semantic checks (e.g., type checking, declaration validation). It owns references to essential components used during semantic analysis:

Main members inside the `Sema`:

* `ASTContext` – manages all AST nodes and semantic info
* `PreProcessor` – token stream handler
* `LangOptions` – holds active language dialect flags
* `DiagnosticsEngine` – emits warnings and errors
* `SourceManager` – tracks source locations

### `Scope`

Tracks the current scope during parsing and semantic analysis. It helps `Sema` resolve names (variables, functions, types) correctly depending on the nesting level (e.g., inside functions, loops, conditionals, or namespaces).

For example:

```c
int foo() {
    while (1) {
        if (bar) {
            foobar();
        }
    }
}
```

`foobar` is in the `if` scope, which is in the `while` scope, which is in the function scope.

### `Decl`

`Decl` is the base class for all AST nodes representing C/C++ declarations (e.g., functions, variables, classes).
All declaration nodes (like `FunctionDecl`, `VarDecl`, `RecordDecl`) derive from `Decl`.

`DeclGroupRef` is a container used when multiple declarations are parsed together (e.g., `int a, b;`).

### `ASTContext`

`ASTContext` manages the lifetime and storage of all AST nodes and semantic information. It contains information about:

Main members inside the `ASTContext`:

* `Decl` nodes (all AST nodes)
* `LangOpts`
* `TargetInfo`

### `ASTConsumer`

A callback interface class to observe and process AST nodes as they're built.

### `Parser::ModuleImportState`

An enum only for C++20 `module`/`import` that can only be placed at the start of a file. After the first declaration, the keywords `module`/`import` become normal identifiers that you can use.

Example:

```cpp
module;
import foo; // valid here
int x;
import bar; // now invalid
int import = 1; // valid
```

# parsing logic overview

## clang::ParseAST

* Sets up the statistics system (used for diagnostics and performance tracking)
* Initializes `Sema`
* Creates the `ASTConsumer` from `Sema`, which will receive the AST nodes
* Constructs the `Parser`, connecting the `Preprocessor` and `Sema`
* Registers crash recovery cleanup routines
* Ensures the lexer is available
* Initializes the parsing logic — this is where the first token is read and the first scope is set (`DeclScope`)
* Main parsing logic happens in `HandleTopLevelDecl`, which in your case emits LLVM IR
* Processes things like `#pragma weak`
* create the Target (ex `.o`)
* Prints stats

### parsing logic

This is the main parsing logic of the Clang `Parser`.

```cpp
    Parser::DeclGroupPtrTy ADecl; // temp for the current Decl
    Sema::ModuleImportState ImportState; // import state for C++20

    for (bool AtEOF = P.ParseFirstTopLevelDecl(ADecl, ImportState); !AtEOF;
         AtEOF = P.ParseTopLevelDecl(ADecl, ImportState)) {

      if (ADecl && !Consumer->HandleTopLevelDecl(ADecl.get()))
        return;
    }
```

`ParseFirstTopLevelDecl` is a wrapper around `ParseTopLevelDecl` that initializes the `ImportState` for C++20. So this loop returns a `Decl` and while it's not `AtEOF`, the `Decl` is passed to `Consumer->HandleTopLevelDecl`, which is the `ASTConsumer` that transforms it to the target — in your case, IR.

`Parser::DeclGroupPtrTy` is a pointer to a `DeclGroupRef`, which is an interface for [these classes](clang%20Decltype.md).

## Parser::ParseTopLevelDecl

* Sets up a destructor (RAII) for the parser data
* Giant switch-case for special cases (this is where `tok::eof` returns true)
* Initializes `ParsedAttributes` for GNU-style attributes (`__attribute__((foo))`) and C++11-style (`[[foo]]`)
* Parses the trailing attributes of both types
* Calls `ParseExternalDeclaration`
* Handles the `ImportState` for C++20

## Parser::ParseExternalDeclaration

This function decides whether to redirect to `ParseDeclarationOrFunctionDefinition` or `ParseDeclaration`, and handles special cases like `asm` or `import`/`export`. It can be simplified like this:

```cpp
    if (Tok.isEditorPlaceholder()) {
      ConsumeToken();
      return nullptr;
    }
    if (getLangOpts().IncrementalExtensions &&
        !isDeclarationStatement(/*DisambiguatingWithExpression=*/true))
      return ParseTopLevelStmtDecl();

    if (!SingleDecl)
      return ParseDeclarationOrFunctionDefinition(Attrs, DeclSpecAttrs, DS);

    return Actions.ConvertDeclToDeclGroup(SingleDecl);
```

`ParseTopLevelStmtDecl` is for `clang-repl`, so you can use C like you would with the Python REPL:

```c
 ➜  ~ clang-repl
clang-repl> #include <stdio.h>
clang-repl> int foo = 5;
clang-repl> printf("foo = %d\n", foo);
foo = 5
clang-repl> foo = 2;
clang-repl> printf("foo = %d\n", foo);
foo = 2
```

* `ParseDeclaration`: Parses a declaration only (variables, typedefs, namespaces, inline namespaces, etc.). Used in the big switch-case
* `ParseDeclarationOrFunctionDefinition`: Parses either a declaration or a function body, depending on what follows the declarator

## Parser::ParseDeclarationOrFunctionDefinition

The `ParsingDeclSpec (DS)` passed by `ParseExternalDeclaration` is set to `nullptr`, so we’re not using the `if (DS)` part.

But first, what is `ParsingDeclSpec`?

### ParsingDeclSpec

As stated in the class definition, `ParsingDeclSpec` is for parsing a `DeclSpec`:

```cpp
/// A class for parsing a DeclSpec.
class ParsingDeclSpec : public DeclSpec {
```

The constructor of `ParsingDeclSpec` calls the parser’s `getAttrFactory()`, providing what `DeclSpec` needs for initialization. The key advantage of `ParsingDeclSpec` is that it creates a `ParsingDeclRAIIObject` to manage diagnostics — accumulating warnings or errors during parsing and ensuring they’re either committed or discarded when the object goes out of scope.

### DeclSpec

But what is `DeclSpec`?

`DeclSpec` captures all the information about declaration specifiers:

* **Storage specifiers (`SCS`)**: `typedef`, `extern`, `static`, `auto`, `register`, etc.
* **Thread storage specifiers (`TSCS`)**: `__thread`, `thread_local`, `_Thread_local`
* **Type qualifiers (`TQ`)**: `const`, `volatile`, `restrict`, `atomic`, etc.
* **Attribute**

Each category is stored in a compact bitfield along with source-location metadata, allowing Clang to validate and diagnose specifier usage precisely.

Back to `ParseDeclarationOrFunctionDefinition`:

* Initializes a `ObjCDeclContextSwitch` — we don’t care for this since it’s Objective-C
* Enters `ParseDeclOrFunctionDefInternal`

## Parser::ParseDeclOrFunctionDefInternal

* Adds `DeclSpecAttrs` (`__attribute__`) list to the `ParsingDeclSpec`
* Prepares for MS-specific parsing
* Parses freestanding declaration specifiers (`typedef`, `extern`, `static`, `auto`, `register`, `class {}`, `struct {}`, `enum {}`) and fills `DeclSpec`
* Checks for a missing `;` and parses trailing args like `attribute`
* If a `;` is found:

  * Returns the keyword size for diagnostics in `LengthOfTSTToken`
  * Suggests corrections (like fixing `[[attrib]] struct` → `struct [[attrib]]`)
  * Adjusts attribute position if possible
  * Consumes the `;`
  * Creates the AST (`Sema::ParsedFreeStandingDeclSpec`)
  * If it's a `RecordDecl` (struct/union/class), validates it
  * Creates a `DeclGroupPtrTy`
* If a struct has been parsed but is not freestanding (`struct S { … } x;`), notify Sema before parsing `x`
* Handles attributes before Objective-C keywords like `@interface`
* `DS.abort()` — RAII cleanup?
* Adds `Attrs` (`[[attribute]]`) to `DeclSpec`
* Detects `extern "C"`
* Adjusts `[[attribute]]` position if possible
* Calls `ParseDeclGroup` to parse the function and return the result

Functions to detail later:

* `ParseDeclarationSpecifiers`
* `DiagnoseMissingSemiAfterTagDefinition`
* `ParsedFreeStandingDeclSpec`
* `ActOnDefinedDeclarationSpecifier`

## `Parser::ParseDeclGroup`

example of the current state of the compiler

```c
int [[hidden]] x = 5;
```

already consumed `int [[hidden]]`
current Token `x`

```c
static inline void foo() {
    return;
}
```

already consumed `static inline void`
current Token `foo`

function explanation:

* init the `ParsedAttribute` and `ParsingDeclarator` with current data
* init `SuppressAccessChecks`, a RAII class to delay errors for templates of private class members `template <> foo<bar::foobar>`
* `ParseDeclarator` (ex: `int x = 5;` current `;`, `int main(int ac, char av) {}` current: `{`)
* pop out `SuppressAccessChecks`
* if no name parsed, skip until a good place to continue the parsing and return nullptr

```cpp
int /*missing name*/;  
```

* shader-specific parsing
* parse end of `requires` (C++20)

```cpp
template<typename T>
void f(T) requires std::is_integral_v<T>;  
// The 'requires std::is_integral_v<T>' is parsed here
```

* if we are parsing a function

  * parse trailing GNU `__attribute__`
  * if the current token is `_Noreturn`, create a diagnostic: this can't happen
  * if tok == `=` and next token is the code completion of the LSP

  ```cpp
  struct S {
    S() = /*<cursor here>*/a
  };
  ```

  * handle `virtual`/`override` (C++11)

  ```cpp
  struct C { void f(); };
  // Out-of-line definition with invalid 'override'
  void C::f() override { /*...*/ }
  ```

  * determine if this is a function body or prototype; if it's a function def, enter the if

    * file-scope only
    * handle explicit instantiation vs specialization
    * call `ParseFunctionDefinition(...)`
    * return converted DeclGroup

  ```cpp
  int f(int x) { return x*2; } // function def

  int f(int x);  // prototype
  ```

* consume any attribute after declarator and attach them to `DeclaratorDecl`

* handle C++ range-based for loops (and Obj-C for-in)

```cpp
for (auto &x : vec) {
  // ...
}
```

* `declsIncGroup` to explain
* while it's not a comma, continue

  * if this is a new line and in this context this can't be a declarator: diag missing semi
  * error on multiple template declarators
  * `D.clear(); D.setCommaLoc()`
  * parse `__attribute__` again...
  * if (MicrosoftExt) skip MS‐style attrs
  * shader parsing
* if that's a valid type

  * parse `requires`

  ```cpp
  template<typename T>
  void f(),                 // first declarator: no requires
  g() requires C<T>;   // second declarator: has a trailing requires-clause
  ```
* get token location into `DeclEnd`
* if semi and `expectedSemi`

  * if any specifier, skip it
* `Actions.FinalizeDeclaratorGroup`


## **Parser::ParseFunctionDefinition**

* Microsoft-specific SEH (`__try`, `__except`, `__finally`) is flagged as illegal in the function body after parsing
* If compiling C and not in C89, warn about a missing return identifier (this was allowed in older C)
* Handles K\&R-style identifiers:

```c
int sum(a, b)     // ← parameter names only
int a;            // ← types declared separately
int b;
{
    return a + b;
}
```

* Checks if the token is valid (`{`, or in C++ only: `try`, `:`, `=`); otherwise, emit a diagnostic and skip until `{`; if not found, return `nullptr`
* If `=`, check for invalid `attribute`
* If delayed template parsing is enabled and it's a template definition (not `= default` or `= delete`):

  * Only parse the signature, cache the body tokens, and delay full parsing until later
* Else if Objective-C-specific
* Set `parsescope` to:

```c
int foo() {}        // FnScope

struct mytype { };  // DeclScope

void foo() {
  {                 // CompoundStmtScope
    int x = 3;  
  }
}
```

* Parse C++ `= delete` or `= default` as `ActOnStartOfFunctionDef`
* Take a snapshot of the FPF (Float Precision Feature) to restore it later:

```c
void f() {
  #pragma float_control(precise, on)
  float x = a + b;  // precise mode
  
  {
    #pragma float_control(precise, off)
    float y = c + d;  // fast-math mode
  }

  float z = e + f;  // still in precise mode — outer setting restored
}
```

* Sema checks if the base of the parsing is correct
* Skip what should be skipped for delayed template parsing
* Clear the `ParsingDeclarator` RAII before parsing the body
* Finish parsing for special states (e.g., `= default`)
* If using `auto`, transform it into a template by adding depth to the depth traker (this is normaly done by template <> and need to be done manulay for auto f(auto foo) in cpp 20)
* If skipping the function body, finish parsing and return it
* Parse `try`
* If `:`, parse `constructor`
* Parse late `__attribute__`
* Return `ParseFunctionStatementBody`

## **Parser::ParseFunctionStatementBody**

* Save the `{` location for diagnostics
* Setup RAII for C++ method pragma internal state (mainly for Windows)
* Main parsing function: `ParseCompoundStatementBody`
* If the function body is invalid, create a bogus `CompoundStmt`
* Exit the body scope
* Return the finished parsed body

## **Parser::ParseCompoundStatementBody**

* Create the feature RAII
* Start with pragmas
* Parse `__label__` (can only be at the start of a scope)

```c
{
  __label__ retry1;
  __label__ retry2;
  __label__ done1, done2;
}
```

* Set `ParsedStmtContext` to `Compound` | `InStmtExpr`
* While not at the end of the function:

  * Try to parse misplaced C++20 imports
  * Parse redundant semicolons `;;;;;;;;;`
  * If token is not `__extension__`, call `ParseStatementOrDeclaration()`
  * Else, consume all `__extension__` and parse the statement, but silence extension warnings
  * If the last statement is valid, push it to the `Stmts` vector
* Check the last statement
* Emit FP evaluation method warnings for unsupported targets
* Gracefully handle the closing `}` or missing braces
* Build and return a `CompoundStmt` or `StmtExpr` via Sema

## **ParseStatementOrDeclaration**

To be explained

also the sema validation
