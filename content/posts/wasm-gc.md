---
title: 'New types and instructions to be introduced in WasmGC'
publishDate: 2024-01-01
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

We can learn the high-level view of WasmGC in the blog post by V8: [A new way to bring garbage collected programming languages efficiently to WebAssembly · V8](https://v8.dev/blog/wasm-gc-porting)

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

![](/images/type-hierarchy.png)


## Subtypes

```lisp
(type $A (struct)) ;; Abbreviation of `(type $A (sub (struct)))`
(type $A (sub (struct))) ;; class A {}
 ;; Define a type B with a field of i32, that is a subtype of A
(type $B (sub $A (struct (field i32))))
 ;; Define a type C with fields of i32 and i64, that is a subtype of B
(type $C (sub final $B (struct (field i32 i64))))
```

> the preexisting syntax with no sub clause is redefined to be a shorthand for a sub clause with empty typeidx list

## Abbreviations

```lisp
funcref == (ref null func)
externref == (ref null extern)
anyref == (ref null any)
nullref == (ref null none)
nullexternref == (ref null noextern)
nullfuncref == (ref null nofunc)
eqref == (ref null eq)
structref == (ref null struct)
arrayref == (ref null array)
i31ref == (ref null i31)
```


## New instructions

### i31.*

`i31.get_u`, `i31.get_s` and so on (get_u/s are with or without sign extension)

### array.* and struct.*

- `array.new`, `struct.new`
- `array.get/set`, `struct.get/set`
- `array.len/fill/copy`
- and more

```lisp
(type $vector (array (mut f64)))
(type $tup (struct i64 i64 i32))

;; array.new <type> <values>
;; struct.new <type> <values>
(local.set $v (array.new $vector (f64.const 1) (i32.const 3)))
(local.set $t (struct.new $tup (i64.const 1) (i64.const 2) (i64.const 1)))

;; array.get <type> <arrayref> <index>
;; struct.get <type> <index> <structref>
(array.get $vector (local.get $v) (i32.const 1)) ;; Get the 1st element of $v(0-index)
(struct.get $tup 1 (local.get $t)) ;; $t の1番目の要素を取得

;; array.set <type> <arrayref> <index> <value>
;; struct.set <type> <index> <sturctref> <value>
(array.set $vector (local.get $v) (i32.const 2) (i32.const 5)) ;; Set 5 to the 2nd element of $v
(struct.set $tup 1 (local.get $t) (i64.const 100)) ;; Set 100 to the 1st field of $t
```

### br_on_*

Branching based on the value of the reference type

- `br_on_cast`
  - ``(br_on_cast $label <value> <rtt>)``
  - If the runtime type of the second argument `value` can be cast to the `rtt`, push the value of type `rtt` to the stach and branch to `$label`.
- `br_on_cast_fail`
  - ``(br_on_cast_fail $label <value> <rtt>)``
  - If the `value` cannot be cast to `rtt`, branch to `$label` with `value` on the stack.
- `br_on_null $l <value>`
  - ``(br_on_null $l (local.get $r))``
  - If `$r` is a null reference, branch to `$l`
- `br_on_non_null $l`


### ref.*

casting and testing values of reference types

- `ref.null ht` - create a null reference of type `ht` ``(e.g. ref.null eq)``
- `ref.i31 $x` - create `i31ref` from `$x: i32`
- `ref.test <ref ht> <runtime-type>`
    - Checks if the first argument reference can be cast to the second argument runtime-type. 1 if it can, 0 if it cannot.
- `ref.cast <ref ht> <runtime-type>`
  - Casts the first argument reference to the second argument runtime-type.
  - If it cannot be cast, trap
- `ref.is_null <ref ht>`
  - 1 if the given reference is null, 0 otherwise
- `ref.as_non_null <ref null ht>` - make a nullable reference non-nullable? Not sure.
- `ref.eq <ref ht> <ref ht>`
  - Receives two `eqref` operands and performs an identity check on the values of the two references. (I haven't found any mention of specific identity checks).
- `ref.func` - create a function reference of the given function

### extern.*

"converts an external value into the internal representation"

- `extern.convert_any`
- `any.convert_extern`


## Convert between wasm and wat of wasmgc / waml

### Dockerfile

[tanishiking/waml-docker: Dockerfile for build and running WAML](https://github.com/tanishiking/waml-docker)

```sh
$ docker build . -t waml
$ docker run -it waml waml -x -c
waml 0.2 interpreter
> val f x = x + 7;  f 5;
...

# compile waml to wasm
$ docker run -i -v "$PWD":/data waml waml -c /data/test.waml

# compile waml to wat
$ docker run -i -v "$PWD":/data waml waml -c -x /data/test.waml

# convert wat to wasm
$ docker run -i -v "$PWD":/data waml wasm -d /data/test.wat -o /data/test.wasm

# interpret wasm
$ docker run -i -v "$PWD":/data waml wasm /data/test.wasm
```

### wasm reference interpreter

There is a wasm reference interpreter in [the /interpreter directory of the WebAssembly spec](https://github.com/WebAssembly/gc/tree/main/interpreter), and with `WebAssembly/gc` you can use a reference interpreter that supports wasm gc.

V8 can also run wasmgc by default, so you can run it from Deno, Node or other browsers.

### waml

> An experimental functional language and implementation for exploring and evaluating the Wasm GC proposal.

There is also a mini ML language that can be compiled into wasmgc. It is also a good way to learn what kind of code should be converted into what kind of wasmgc code.

Also, read the kotlin/wasm generated wasmgc code [Exploring WAT Files Generated from Kotlin/Wasm | Rikito Taniguchi](https://tanishiking.github.io/posts/kotlin-wasm-deep-dive/)

## References

- [gc/proposals/gc/Overview.md at main · WebAssembly/gc](https://github.com/WebAssembly/gc/blob/main/proposals/gc/Overview.md)
- [gc/proposals/gc/MVP.md at main · WebAssembly/gc](https://github.com/WebAssembly/gc/blob/main/proposals/gc/MVP.md)
- [A new way to bring garbage collected programming languages efficiently to WebAssembly · V8](https://v8.dev/blog/wasm-gc-porting)
- [Wasm GC: What Exactly Is It (and Why I Should Care) - Ivan Mikushin, VMware - YouTube](https://www.youtube.com/watch?v=ndJP-vmZFYk)
- [webassembly - Why do we need the type of i31 in WasmGC proposal? - Stack Overflow](https://stackoverflow.com/questions/77468063/why-do-we-need-the-type-of-i31-in-wasmgc-proposal)
- [Bytecode Alliance — WebAssembly Reference Types in Wasmtime](https://bytecodealliance.org/articles/reference-types-in-wasmtime)
