---
title: 'Learn new types and instructions to be introduced in WasmGC'
publishDate: 2024-12-20
author: 'Rikito Taniguchi'
tags:
  - webassembly
---

In November 2023, the WasmGC[^wasmgc] became the default in Chrome[^chrome] and Firefox[^firefox]. It is a really exciting change, and is a good sign that the WasmGC is coming to standarization.
Some GC programming languages such as OCaml[^ocaml], Kotlin[^kotlin], Dart[^dart], and Java[^j2cl] have an implementation that compiles to Wasm with WasmGC support.


[^wasmgc]: https://github.com/WebAssembly/gc
[^chrome]: https://developer.chrome.com/blog/wasmgc
[^firefox]: https://www.mozilla.org/en-US/firefox/120.0/releasenotes/

[^ocaml]: https://github.com/ocaml-wasm/wasm_of_ocaml
[^kotlin]: https://kotlinlang.org/docs/wasm-overview.html
[^dart]: https://github.com/dart-lang/sdk/blob/5510ef63a6dcd06423305e15e2014cf20ac94699/pkg/dart2wasm/README.md
[^j2cl]: https://github.com/google/j2cl/blob/b3ff9e233cef3e67e495476da14f13250a56c5e9/docs/getting-started-j2wasm.md

However, what exactly is WasmGC? What compilers need to do to support it?

From compilers' point of view, WasmGC is basically an extension of Wasm language with new heap types, and new instruction to allocate and control those heap objects. If compilers generate Wasm code with those new WasmGC types and instruction (WasmGC primitives), then the Wasm code can be executed with GC, in the runtime that supports WasmGC.

In this article, we will explore the new types and instructions that will be introduced by WasmGC.

## Reference Types

```wasm
(type $A (struct)) ;; index = 0
(func $f1 (param (ref null $A))) ;; func f1(a *A) { ... }
(func $f2 (result (ref 0))) ;; func f2() *A { ... }
```
