#llvm
# Tester

run all check
```
ninja check 
```
* `-k <n>`: stop after n test (if n == 0 this will never stop)

## **Unit Tests**

   * **Purpose:** Test individual C++ components and APIs
   * **Location:** `llvm/unittests/`
   * **Framework:** Google Test (`gtest`)
   * **Run with:** `ninja check-llvm-unit`

## **Integration/Regression Tests**

   * **Purpose:** Validate IR transformations, optimizations, codegen, diagnostics
   * **Location:** `llvm/test/`, `clang/test/`, etc.
   * **Run with:** `ninja check-<project>`
   * **Tools:**

## lit (LLVM Integrated Tester)
runs tests that are all the comment RUN (`; RUN`)

* `RUN <cmd>`: run a test
* `REQUIRE <config>`: run test only if the requirement are fill

### Example
```
; RUN: opt -S -passes=instcombine < %s | FileCheck %s
```

### Substitutions

* `%s`: Source file
* `%S`: Source folder
* `%t`: Path to a unique temp file

### Usage
```sh
llvm-lit <test or tests folder>
```
* `-s`: silent print
* `-v`: print the `RUN` line and output test on failure
* `-a`: like `-v` but all the time
* `-k <n>`: stop after n test (if n == 0 this will never stop)

## `FileCheck`

checks compiler output against patterns in `; CHECK:` lines

* `CHECK:`: match a line exactly
* `CHECK-NEXT:`: match the next line
* `CHECK-NOT:`: assert a line doesn't appear between matches
* `CHECK-DAG:`: match a set of lines in any order, without intervening lines
* `CHECK-LABEL:`: reset matching at a specific anchor

### Example

```llvm
; CHECK-LABEL: @my_function
; CHECK: %x = add i32 %a, %b
; CHECK-NEXT: ret i32 %x
```

### Match Variables

* `[[VAR:regex]]`: capture a value with a regex
* `%[[VAR]]`: reuse the captured value

### Usage

```sh
opt -S -passes=instcombine < input.ll | FileCheck input.ll
```
