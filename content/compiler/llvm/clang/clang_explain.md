#clang #llvm 
# **clang frontend "happy path"**

I welcome you to this walkthrough of the clang frontend execution!!! this walkthrough is mainly for me to dive into the clang source code, but if it helps others along the way, that’s a win!!!

**note:** to get the most out of this guide, follow along with the source code open at your side and spot what’s happening in real time.
**note:** i won't go into detail on `logging`, `error handling`, `windows support`, or `objective-c specifics` — this is mainly focused on C to avoid some C++ overhead like delayed template parsing details.

I recommend watching these for a great overview of the clang frontend:

* [2019 LLVM Developers’ Meeting: S. Haastregt & A. Stulova “An overview of Clang”](https://www.youtube.com/watch?v=5kkMpJpIGYU)
* [The Clang AST - a tutorial](https://www.youtube.com/watch?v=VqCkCDFLSsc)

I separated the explanation into 3 parts to make it a bit more enjoyable:

* [The Clang Driver](clang_driver.md)
* [The Clang Compiler Parsing/AST](clang_cc1.md)
* [The Clang Compiler Link to LLVM](clang_link_backend.md)
