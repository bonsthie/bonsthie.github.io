#clang #llvm 

| **Class**                | **Represents**                                                                |
| ------------------------ | ----------------------------------------------------------------------------- |
| `TagDecl`                | Abstract base for C/C++ tag declarations (`struct`, `union`, `class`, `enum`) |
| `RecordDecl`             | C `struct` or `union` definition                                              |
| `CXXRecordDecl`          | C++ `class` or `struct` definition (with methods, bases)                      |
| `EnumDecl`               | C/C++ `enum` definition                                                       |
| `EnumConstantDecl`       | Single enumerator inside an `EnumDecl`                                        |
| `TypedefNameDecl`        | C `typedef` declaration                                                       |
| `TypeAliasDecl`          | C++11 `using Name = ...;` type alias                                          |
| `VarDecl`                | Variable declaration (global, local, static, extern)                          |
| `FieldDecl`              | Non-static data member in a record/class                                      |
| `FunctionDecl`           | Free or static member function                                                |
| `CXXMethodDecl`          | C++ non-static member function                                                |
| `CXXConstructorDecl`     | C++ constructor                                                               |
| `CXXDestructorDecl`      | C++ destructor                                                                |
| `CXXConversionDecl`      | C++ conversion operator                                                       |
| `ParmVarDecl`            | Function parameter                                                            |
| `LinkageSpecDecl`        | `extern "C"` or `extern "C++"` linkage region                                 |
| `NamespaceDecl`          | C++ `namespace { ... }`                                                       |
| `NamespaceAliasDecl`     | C++ `namespace A = B;` alias                                                  |
| `UsingDirectiveDecl`     | C++ `using namespace N;` directive                                            |
| `UsingDecl`              | C++ `using N::X;` declaration                                                 |
| `FunctionTemplateDecl`   | C++ function template declaration                                             |
| `ClassTemplateDecl`      | C++ class template declaration                                                |
| `AccessSpecDecl`         | C++ `public:`, `protected:`, `private:` label in class                        |
| `ObjCInterfaceDecl`      | Objective-C `@interface` definition                                           |
| `ObjCCategoryDecl`       | Objective-C `@interface MyClass(Category) { ... }`                            |
| `ObjCProtocolDecl`       | Objective-C `@protocol` definition                                            |
| `ObjCImplementationDecl` | Objective-C `@implementation` definition                                      |
| `ImportDecl`             | `#import` or `@import` of a module/header                                     |
| `StaticAssertDecl`       | C++17 `static_assert(...)` declaration                                        |
| `FriendDecl`             | C++ `friend` declaration inside a class                                       |
| `LabelDecl`              | Label for `goto`                                                              |
| `CapturedDecl`           | Lambda/coroutine captured variables                                           |
| `BlockDecl`              | Objective-C block literal function declaration                                |
