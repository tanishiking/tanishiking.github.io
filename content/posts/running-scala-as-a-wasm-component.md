---
title: "Running Scala as a Wasm Component"
description: "A note on where Scala on WebAssembly Components is today: scala-wasm, WIT bindings, Scala/Rust component composition, Spin server applications, and some current limitations."
publishDate: 2026-06-06
draft: true
author: 'Rikito Taniguchi'
tags:
  - Scala
  - Scala.js
  - WebAssembly
  - Wasm
  - Spin
---

## Summary

- [Scala.js](https://www.scala-js.org/) has a Wasm backend, but for now it assumes a JavaScript host.
- [scala-wasm](https://github.com/scala-wasm/scala-wasm) is a fork where we are experimenting with Wasm modules and [Wasm Component Model](https://component-model.bytecodealliance.org/) support without JavaScript. This makes a few things possible:
  - running programs safely in a sandbox, such as AI-generated code or user-uploaded untrusted code
  - interoperability between components written in different languages, with better protection against supply chain attacks
  - very fast cold starts.
- [scala-wasm/component-example](https://github.com/scala-wasm/component-example) has examples of Scala + Component Model usage.
- There are also server-side examples/templates with [Spin](https://spinframework.dev/) in [scala-wasm/scala-wasm-spin-templates](https://github.com/scala-wasm/scala-wasm-spin-templates).
- Limitations:
  - Libraries depending on JavaScript cannot be used as-is. They need a WASI-oriented port. Experimental [cats-effect + FS2 WASI work](https://summerofcode.withgoogle.com/programs/2026/projects/vibpm0UF) is in progress.
  - GC+EH binaries are not supported by [SpinKube](https://www.spinkube.dev/) or [Akamai Functions](https://www.akamai.com/products/akamai-functions) yet. [SpinKube support is being worked on](https://github.com/spinframework/containerd-shim-spin/pull/444).

## Introduction

### Wasm, WASI, and Wasm Components

Recently, Wasm is used in different areas: safely running AI-generated code, server-side applications with ultra-fast cold starts, and language-independent plugin systems.

What Wasm itself can do is just a pure computation. It cannot affect the outside world directly. Things like file I/O, clocks, and networking have to go through interfaces imported from the host.

So unless the host explicitly gives it access, a Wasm module cannot perform external effects. This is why Wasm is often described as a sandbox.

For example, the following Wasm module does not implement `log` by itself. `run` can have an external effect only if the host environment (like JS) passes an import for `log`. Filesystems, clocks, networking, and so on work in the same way: the host has to provide those interfaces explicitly.

```wat
(module
  (import "env" "log" (func $log (param i32)))
  (func (export "run")
    i32.const 42
    call $log))
```

```js
const imports = {
  env: {
    log: value => console.log(value),
  },
};
const { instance } = await WebAssembly.instantiate(bytes, imports);
instance.exports.run(); // 42
```

[WASI](https://wasi.dev/) is an effort to standardize these system interfaces. The current version, WASI Preview 2 (and 3), is defined on top of the [Wasm Component Model](https://component-model.bytecodealliance.org/).

The Wasm Component Model gives components typed imports and exports through interfaces written in [WIT](https://component-model.bytecodealliance.org/design/wit.html) interface description language. With Component Model, components written in different languages can be composed without depending on the ABI or runtime representation of each different language.

```wit
// Example of WIT
package example:greeter;
interface greeter {
  greet: func(name: string) -> string;
}
world app {
  import greeter;
}
```

Wasm Components also use a [shared-nothing architecture](https://github.com/WebAssembly/component-model/blob/823aa133c2d94a328269da25fd9712bccc142d49/design/high-level/Choices.md). One component cannot directly access outside worlds' memory or data. Communication has to go through WIT-defined interfaces. This helps keep the impact of a malicious dependency inside component boundaries, which is good for supply chain security.

I will not go into the details of WASI or the Component Model here. These documents are good starting points:

- [Why the Component Model?](https://component-model.bytecodealliance.org/design/why-component-model.html)
- [Component Model Concepts](https://component-model.bytecodealliance.org/design/component-model-concepts.html)
- [WIT Reference](https://component-model.bytecodealliance.org/design/wit.html)
- [WASI.dev](https://wasi.dev/)

### Scala.js and Wasm

[Scala.js has supported a Wasm backend since 1.17.0](https://www.scala-js.org/news/2024/09/28/announcing-scalajs-1.17.0/), so Scala.js programs can be compiled to Wasm modules. However, the current Wasm backend assumes a JavaScript environment such as Node.js or a browser. It cannot directly target Wasm runtimes without JavaScript, such as `wasmtime`, nor the Wasm Component Model.

To explore that area, we are experimenting in a Scala.js fork called `scala-wasm`. The fork supports Wasm modules and Wasm Components without JavaScript.

The goal of this fork is to eventually upstream this work into Scala.js. Actually, as a first step, we are working on upstreaming [Minimal Wasm](https://github.com/scala-js/scala-js/pull/5353), a feature for generating Wasm modules without JavaScript.

https://github.com/scala-wasm/scala-wasm

In this article, I will summarize where Scala and Wasm are today, and show how to compile Scala code into Wasm Components with `scala-wasm`.

## Compiling Scala to Wasm Components

Let's compile Scala code to Wasm Components with `scala-wasm`.

> [!Disclaimer]
> `scala-wasm` is an experimental project. APIs may change significantly across versions.

I will cover Scala/Rust interoperability and a TODO-list server example using [Spin](https://spinframework.dev/), a Wasm framework. For complete code and more examples, see [scala-wasm/component-example](https://github.com/scala-wasm/component-example).

### Requirements

- [wasm-tools](https://github.com/bytecodealliance/wasm-tools)
- [wasmtime](https://github.com/bytecodealliance/wasmtime)
- [scala-wasm/wit-bindgen](https://github.com/scala-wasm/wit-bindgen)
  - a scala-wasm-compatible fork
- [wkg](https://github.com/bytecodealliance/wasm-pkg-tools)
- [wac](https://github.com/bytecodealliance/wac)
- [cargo-component](https://github.com/bytecodealliance/cargo-component)
- [Spin canary](https://developer.fermyon.com/spin/v4/install)

The `wit-bindgen` version compatible with `scala-wasm` has not yet been upstreamed to `bytecodealliance/wit-bindgen`, so install it from the fork:

```sh
$ cargo install --git https://github.com/scala-wasm/wit-bindgen --tag scala-wasm-wasm.4
```

### sbt plugin

`scala-wasm` is published under the `io.github.scala-wasm` organization. Add this to `project/plugins.sbt`:

```scala
addSbtPlugin("io.github.scala-wasm" % "sbt-scalajs" % "1.21.1-wasm.4")
libraryDependencies += "io.github.scala-wasm" %% "scalajs-env-wasmtime" % "0.0.2"
```

In `build.sbt`, set `scalaJSLinkerConfig`'s `moduleKind` to `WasmComponent`. Then the linker produces a Wasm Component. If there are WIT files under `scalaJSWitDirectory`, `wit-bindgen scala` runs at compile time and generates Scala bindings under `target/scala-*/src_managed`.

```scala
import org.scalajs.linker.interface.ModuleKind
enablePlugins(ScalaJSPlugin)

scalaJSWitDirectory := baseDirectory.value / "wit"
scalaJSWitWorld := Some("world-name") // WIT world to generate

scalaJSLinkerConfig ~= {
  val witDir = scalaJSWitDirectory.value
  val witWorld = scalaJSWitWorld.value
  _.withExperimentalUseWebAssembly(true)
    .withModuleKind(ModuleKind.WasmComponent)
    .withWasmFeatures { features =>
      // This currently needs to be passed through wasmFeatures as well.
      // Ideally this should become unnecessary in the future.
      features
        .withWitDirectory(Some(witDir.getAbsolutePath))
        .withWitWorld(witWorld)
    }
}
```

## Scala + Rust Interoperability with Wasm Components

First, let's create a project where Scala and Rust call each other through Wasm Components. The example is simple: Scala calls a `greeter` interface implemented in Rust.

I will only show some snippets here. The full code is in [component-example/rust-compose](https://github.com/scala-wasm/component-example/tree/b7ebc627447a57b2bc80e7fbcbfff278f3edfa2b/rust-compose).

```wit
package scala-wasm:rust-compose;

interface greeter {
  greet: func(name: string) -> string;
}
world scala {
  export wasi:cli/run@0.2.0;
  import greeter;
}
world rust {
  export greeter;
}
```

First, implement the Scala component for the `scala` world.

```scala
import org.scalajs.linker.interface.ModuleKind
lazy val rustComposeScala = project
  .settings(
    // ...
    scalaJSWitDirectory := baseDirectory.value / "wit",
    scalaJSWitWorld := Some("scala"), // implement the scala world
    scalaJSWitPackage := Some("rustcompose"), // package for Scala bindings
    Compile / scalaJSLinkerConfig := {
      // ...
      (Compile / scalaJSLinkerConfig).value
        .withExperimentalUseWebAssembly(true)
        .withModuleKind(ModuleKind.WasmComponent)
        // ...
    }
  )
```

Then we can use the generated bindings in managed sources to call `greeter.greet`.

```scala
package rustcompose

import scala.scalajs.wit
import scala.scalajs.wit.annotation._

import rustcompose.exports.wasi.cli.Run
import rustcompose.scala_wasm.rust_compose.greeter.greet

@WitImplementation
object RunImpl extends Run {
  // The WIT type of wasi:cli/run is `result<unit, unit>`.
  override def run(): wit.Result[Unit, Unit] = {
    println(greet("Scala")) // call greet and print the result
    new wit.Ok(())
  }
}
```

On the Rust side, generate bindings from the same WIT with `cargo component`, and implement the `greeter` interface. Here we pass the received string to [ferris-says](https://github.com/rust-lang/ferris-says), and return the resulting string.

```rust
#[allow(warnings)]
mod bindings;
use crate::bindings::exports::scala_wasm::rust_compose::greeter::Guest;
struct Component;
impl Guest for Component {
    fn greet(name: String) -> String {
        let mut buffer = Vec::new();
        let message = format!("Hello from Rust, {name}!");
        ferris_says::say(&message, 80, &mut buffer).unwrap();
        String::from_utf8(buffer).unwrap()
    }
}
bindings::export!(Component with_types_in bindings);
```

After building both components, use `wac plug` to compose the Rust component into the Scala component's import.

```sh
$ wkg wit fetch
$ sbt rustComposeScala/fastLinkJS
$ cd rust-compose/rust-greeter
$ cargo component build --target wasm32-wasip2 -r
$ cd ..
$ wac plug \
  --plug rust-greeter/target/wasm32-wasip1/release/rust_greeter.wasm \
  scala/target/scala-2.13/rust-compose-scala-fastopt/main.wasm \
  -o out.wasm
$ wasmtime -W gc,function-references,exceptions out.wasm
 _________________________
< Hello from Rust, Scala! >
 -------------------------
        \
         \
            _~^~^~_
        \) /  o o  \ (/
          '_   -   _'
          / '-----' \
```

From Scala's point of view, this looks like a normal function call. Under the hood, it crosses a WIT-typed component boundary and calls the Rust implementation.

## Running an HTTP + SQLite Component with Spin

Next, let's implement a server application as a Wasm Component.

Wasm/WASI behaves like a sandbox: it cannot access external resources unless explicit permissions are granted. Therefore, Wasm can be executed directly on Kubernetes without going through a userland layer of Linux container. This makes cold starts extremely fast.

So how do we implement a practical server application in Wasm? If Wasm cannot freely access the outside world, how do we access databases? One current approach is to use a Wasm framework such as [Spin](https://spinframework.dev/).

Here, we will implement a server-side application in Scala using Spin.
The full code is in [scala-wasm/component-example/spin-todo](https://github.com/scala-wasm/component-example/tree/main/spin-todo). The templates are in [scala-wasm/scala-wasm-spin-templates](https://github.com/scala-wasm/scala-wasm-spin-templates).

One caveat: the Wasm Components emitted by Scala.js use Wasm GC and exception handling features, so Spin canary is currently required.

```sh
$ curl -fsSL https://spinframework.dev/downloads/install.sh | bash -s -- -v canary
```

Create a project from the scala-wasm Spin templates:

```sh
$ spin templates install --git https://github.com/scala-wasm/scala-wasm-spin-templates
$ spin new -t http-scala-wasm-todo spin-todo
$ cd spin-todo
```

The TODO API template generates a component that exports WASI HTTP and imports Spin's SQLite interface.

```wit
package scala-wasm:spin-todo;
world todo {
  import fermyon:spin/sqlite@2.0.0;
  export wasi:http/incoming-handler@0.2.0;
}
```

`wasi:http/incoming-handler` is the interface for handling incoming WASI HTTP requests. A containerd shim for Wasm containers can use it to bridge incoming requests into a Wasm Component.

Build and start the application with `spin up --build`. For now, GC and related Wasm features must be enabled with `--experimental-wasm-feature`.

```sh
$ spin up --build \
  --experimental-wasm-feature gc \
  --experimental-wasm-feature exceptions \
  --experimental-wasm-feature function-references \
  --experimental-wasm-feature reference-types \
  --sqlite @migration.sql
```

Once it starts, you can call the HTTP API:

```sh
$ curl -s http://127.0.0.1:3000/todos
$ curl -s http://127.0.0.1:3000/todos \
  -d '{"title":"Try Scala-wasm on Spin"}'
$ curl -s -X DELETE http://127.0.0.1:3000/todos/1
```

Applications built with Spin can normally be deployed to Kubernetes with [SpinKube](https://www.spinkube.dev/), or to [Akamai Functions](https://www.akamai.com/products/akamai-functions) for Functions at the Edge. However, at the moment, binaries generated by `scala-wasm` cannot be deployed to SpinKube. Scala.js-generated Wasm uses features such as WasmGC and exception handling, and those environments do not support them yet.

SpinKube support is being worked on in [containerd-shim-spin#444](https://github.com/spinframework/containerd-shim-spin/pull/444). For Fermyon Cloud and Akamai Functions, I am looking forward to future support.

## FAQ

### Can I use library XXX?

If the library depends on JavaScript, it cannot be used as-is. Wasm Components are meant to run without a JavaScript runtime. If code paths depending on JS APIs like `scala.scalajs.js.Dynamic` and `@JSImport`, linking should fail with an error like this:

```text
[error] Uses JS interop with a Wasm-only module kind:
[error]   at file:/.../javalib/src/main/scala/java/lang/_String.scala:416:43: this["repeat"](count)
[error]   at file:/.../javalib/src/main/scala/java/lang/_String.scala:429:29: str["substring"](0, (resultLength -[int] str.length;I()))
[error]   dispatched from java.lang.String.repeat(int)java.lang.String
[error]   called from java.util.regex.PatternSyntaxException.getMessage()java.lang.String
```

In particular, libraries depending on system features such as HTTP, filesystems, processes, and sockets cannot be implemented in Wasm alone. They need to be rewritten to use explicitly imported host interfaces, such as WASI HTTP or WASI filesystem.

For target-specific implementation branching like this, use `scala.scalajs.LinkingInfo.linkTimeIf`. `linkTimeIf` is resolved at link time, and the unused branch is dropped from linking. So we can switch between an implementation for a JavaScript host and an implementation for Wasm-without-JS.

```scala
import scala.scalajs.LinkingInfo.{linkTimeIf, moduleKind}
import scala.scalajs.ModuleKind.{MinimalWasmModule, WasmComponent}

def currentTimeMillis(): Long =
  linkTimeIf(moduleKind == WasmComponent) {
    wasiClockCurrentTimeMillis()
  } {
    import scala.scalajs.js
    js.Date.now().toLong
  }
```

In `scala-wasm`'s javalib, this pattern is used in many places, including `System.currentTimeMillis()` and the regular expression engine, so that JS-dependent code is not linked for Wasm Components.

While many system libraries doesn't work for Wasm Components, experimental [cats-effect and FS2 Wasm/WASI ports](https://summerofcode.withgoogle.com/programs/2026/projects/vibpm0UF) are in progress.

### Why not split out Wasm-specific library artifacts?

`linkTimeIf` is similar to conditional compilation in C/C++ or Rust. Another option would be to add a `wasm` compile target at build time, like Scala.js or Scala Native. Then libraries would publish `wasm` artifacts, and dependency resolution could tell whether a library supports Wasm-without-JS.

On the other hand, Scala.js IR for libraries that do not depend on JavaScript can already be used by `scala-wasm` as-is. Splitting artifacts would mean every library has to republish for Wasm-without-JS, even when the existing Scala.js IR would have worked.

### When will this be upstreamed?

The immediate goal is to upstream [Minimal Wasm](https://github.com/scala-js/scala-js/issues/5333) into Scala.js as a stepping stone toward Wasm Component support. Internally, this means upstreaming large changes that remove JavaScript dependencies from JDK APIs and the Wasm backend. After that, the next step will be to upstream Wasm Component API support into Scala.js.
