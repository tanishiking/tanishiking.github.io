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

```lisp
(type $A (struct)) ;; index = 0
(func $f1 (param (ref null $A))) ;; func f1(a *A) { ... }
(func $f2 (result (ref 0))) ;; func f2() *A { ... }
```

`(ref null <heap-type>)` means that the reference can be a null reference.

## Heap Types

The wasm gc proposal adds heap to wasm engine in addition to stack and linear memory (or perhaps it is more accurate to say, a heap is required for the wasmgc execution engine).

> "The introduction of managed data adds new forms of types that describe the layout of memory blocks on the heap"
> https://github.com/WebAssembly/gc/blob/main/proposals/gc/Overview.md#types

### structures

```lisp
(type $time (struct (field i32) (field f64)))
(type $point (struct (field $x f64) (field $y f64) (field $z f64)))
```

- `structtype ::= struct <fieldtype>*`
- `fieldtype ::= <mutability> <storagetype>`

### arrays

```lisp
(type $vector (array (mut f64)))
(type $matrix (array (mut (ref $vector))))
```

Basically the same as structures. `arraytype ::= array <fieldtype>`

### Recursive types

`rec` allows types that are mutually recursive to be defined. And a single type definition can also be recursive.

```lisp
(rec
  (type $A (struct (field $b (ref null $B))))
  (type B$ (struct (field $a (ref null $A))))
)
(type $C (struct field $f i32) (field $c (ref null $C)))
```

### Unboxed Scalars

`i31ref` https://github.com/WebAssembly/gc/blob/main/proposals/gc/Overview.md#unboxed-scalars

In order to treat numeric types such as `i32` in the same way as other heap types (structures and arrays), those types need to be boxed (which can be a costly operation if the value is frequently accessed).
Unboxed scalars allow wasm engine to avoid the boxing cost for frequently accessed primitive types.

Instead of boxing a `i32`, 31 bits are used to store the actual value and the remaining 1 bit is used to flag indicate that "it is not a pointer, and the remaining 31 bits contain the value, so you can use it as it is". This allows 31-bit numeric types to be used in the same way as other reference types, without the cost of boxing.

(IIUC)

- [webassembly - Why do we need the type of i31 in WasmGC proposal? - Stack Overflow](https://stackoverflow.com/questions/77468063/why-do-we-need-the-type-of-i31-in-wasmgc-proposal)o
- [Tagged pointers and object headers - Writing Interpreters in Rust: a Guide](https://rust-hosted-langs.github.io/book/chapter-interp-tagged-ptrs.html)

### External Types

References/functions in the host can be treated as type `externref/funcref` in WASM. The following hello function can accept parameters of type `externref`, e.g. the host JS code can give a reference in JS to hello as an argument.

```lisp
(func (export "hello") (param externref) ...)
```

Refer [Bytecode Alliance — WebAssembly Reference Types in Wasmtime](https://bytecodealliance.org/articles/reference-types-in-wasmtime) for more details.

Also, [type imports proposal](https://github.com/WebAssembly/proposal-type-imports/blob/main/proposals/type-imports/Overview.md) can be interesting.

## Type Hierarchy

The heap types that represent the data layouts placed in a heap can be classified into three types

- Internal (values in Wasm representation)
- External (values in a host-specific representation)
- Functions

And each has a type hierarchy

- `eq` is the common super type of all reference types that can be compared by `ref.eq`
- `any` is the top type of internal types
- `none` is the bottom type of internal types
- `noextern` is the bottom type of external types
- `nofunc` is the bottom type of function types
- `struct` is the super type of all struct types
- `array` is super type of all array types
- `i31` is a supertype of unboxed scalars

in summary
