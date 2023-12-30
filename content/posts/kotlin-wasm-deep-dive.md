---
title: 'Exploring WAT Files Generated from Kotlin/Wasm'
publishDate: 2023-12-21
author: 'Rikito Taniguchi'
tags:
  - webassembly
  - kotlin
---

In this article, I'm examining the generated WAT (WebAssembly Text) files from `Kotlin/Wasm` and investigating how high-level constructs in Kotlin map to WasmGC.

## Generating WAT from Kotlin/Wasm

https://github.com/Kotlin/kotlin-wasm-examples

I'll work with the `kotlin-wasm-example/nodejs-example` and inspecting the output. The version at the time of writing is Kotlin 1.9.20.

To generate WAT files, I'm passing the `-Xwasm-generate-wat` flag to the compiler[^flags].


```diff
❯ git diff nodejs-example/build.gradle.kts
diff --git a/nodejs-example/build.gradle.kts b/nodejs-example/build.gradle.kts
index 6b48777..fa21751 100644
--- a/nodejs-example/build.gradle.kts
+++ b/nodejs-example/build.gradle.kts
@@ -26,3 +26,9 @@ rootProject.the<NodeJsRootExtension>().apply {
 tasks.withType<org.jetbrains.kotlin.gradle.targets.js.npm.tasks.KotlinNpmInstallTask>().configureEach {
     args.add("--ignore-engines")
 }
+
+tasks.withType<org.jetbrains.kotlin.gradle.tasks.Kotlin2JsCompile>().configureEach {
+    kotlinOptions.freeCompilerArgs += listOf(
+        "-Xwasm-generate-wat",
+    )
+}
```

[^flags]: You can find Kotlin compiler options related to Wasm [here](https://github.com/JetBrains/kotlin/blob/8a863e00ba148f47d11c825faffd92c494e52ba6/compiler/cli/cli-common/src/org/jetbrains/kotlin/cli/common/arguments/K2JSCompilerArguments.kt#L594-L651)

I've edited `nodejs-example/src/wasmJsMain/kotlin/Main.kt` as follows for simplifying the build result (since the main function has a lot of glue code, making it hard to read).

```kotlin
fun main() {
    box()
}
fun box() {
    // ...
}
```

```sh
$ cd nodejs-example
$ ./gradlew build
$ cat ./build/js/packages/kotlin-wasm-nodejs-example-wasm-js/kotlin/kotlin-wasm-nodejs-example-wasm-js.wat
```

## Examining the WAT Generated by Kotlin/Wasm
The generated WAT file is around 3000 lines long. I'll extract relevant code fragments for clarity.

### Direct Function Calling
Let's start with the simplest example:


```kotlin
fun foo(a: Int, b: Int) = a + b
fun box() {
    foo(1, 1)
}
```

```wasm
(type $____type_3 (func (param)))
(func $box___fun_62 (type $____type_3)
    i32.const 1
    i32.const 1
    call $foo___fun_61
    drop)

(type $____type_0 (func (param i32 i32) (result i32)))
(func $foo___fun_61 (type $____type_0)
    (param $0_a i32)
    (param $1_b i32) (result i32)
    local.get $0_a  ;; type: kotlin.Int
    local.get $1_b  ;; type: kotlin.Int
    i32.add
    return)
```

Ok, that's easy.


### (data) class


```kotlin
data class Foo(val bar: Int) {
    var baz: Int = 0
}
fun box() {
    val foo = Foo(100)
    foo.baz = 10
}
```



First, the `Person` class is represented with a struct.

```wasm
(type $Foo___type_36 (sub $kotlin.Any___type_13 (struct
  (field (ref $Foo.vtable___type_26))
  (field (ref null struct))
  (field (mut i32))
  (field (mut i32))
  (field (mut i32)) ;; bar
  (field (mut i32))))) ;; baz
```


`$Foo___type_36` is a subtype of `kotlin.Any___type_13`, and `Any` has the following data structure:


```wasm
(type $kotlin.Any___type_13 (struct
    (field (ref $kotlin.Any.vtable___type_12)) ;; vtable
    (field (ref null struct)) ;; itable
    (field (mut i32)) ;; typeInfo
    (field (mut i32)))) ;; hashCode
```

The presentation at WASM/IO 2023 has a great explanation on the data representation: [Introducing Kotlin/Wasm · seb.deleuze.fr](https://seb.deleuze.fr/introducing-kotlin-wasm/)


`vtable` and `itable` are well-known data structures used for dynamic dispatch[^dispatch]. I'll provide more detailed information in the later section on virtual calls.

Next, let's look at the implementation of `box` and how `Foo` is initialized.

[^dispatch]: see https://lukasatkinson.de/2016/dynamic-vs-static-dispatch/ and https://lukasatkinson.de/2018/interface-dispatch/


```wasm
(func $box___fun_62 (type $____type_3)
    (local $0_foo (ref null $Foo___type_36))
    ref.null none
    i32.const 100
    call $Foo.<init>___fun_61
    local.tee $0_foo  ;; type: <root>.Foo
    i32.const 10
    struct.set $Foo___type_36 5  ;; name: baz, type: kotlin.Int)
```

- `val foo = Foo(100)`
  - `ref.null none` (null reference with bottom type as RTT) and `i32.const 100` as operand `call $Foo.<init>___fun_61`
- `foo.baz = 10`
  - with reference to an instance of `Foo` (`$0_foo`) and `i32.const 10` as operand, `struct.set $foo___type_36 5` (set `i32.const 10` to the fifth field (`baz`) in `$0_foo`)

Ok, then, what does `$Foo.<init>___fun_61` do?


```wasm
(func $Foo.<init>___fun_61 (type $____type_88)
    (param $0_<this> (ref null $Foo___type_36))
    (param $1_bar i32) (result (ref null $Foo___type_36))
    
    ;; Object creation prefix
    local.get $0_<this>
    ref.is_null
    if
        
        ;; Any parameters
        global.get $Foo.vtable___g_24
        ref.null struct
        i32.const 452
        i32.const 0
        
        i32.const 0
        i32.const 0
        struct.new $Foo___type_36
        local.set $0_<this>
    end
    
    local.get $0_<this>  ;; type: <root>.Foo
    local.get $1_bar  ;; type: kotlin.Int
    struct.set $Foo___type_36 4  ;; name: bar, type: kotlin.Int
    local.get $0_<this>  ;; type: <root>.Foo
    i32.const 0
    struct.set $Foo___type_36 5  ;; name: baz, type: kotlin.Int
    local.get $0_<this>
    return)
```

- If `$0_<this>` received as an argument is `null`, then
  - Initialises vtable, itable, typeinfo and hashcode
  - `Foo.vtable` is global
- bar and baz are given initial values, and a pointer `$0_<this>` to the Foo is returned

How vtable and itable are created and used is described in the next section.

### virtual call

```kotlin
open class Base(p: Int) {
    open fun foo() { 1 }
}
class Derived(p: Int) : Base(p) {
    override fun foo() { 2 }
}

fun box() {
    val d = Derived(1)
    bar(d)
}
fun bar(f: Base) = f.foo()
```

Base type:


```wasm
;; type definition
(type $Base___type_36 (sub $kotlin.Any___type_13 (struct
    (field (ref $Base.vtable___type_26)) (field (ref null struct)) (field (mut i32)) (field (mut i32)))))
(type $Base.vtable___type_26 (sub $kotlin.Any.vtable___type_12 (struct
    (field (ref null $____type_53)))))
(type $____type_53 (func (param (ref null $kotlin.Any___type_13)) (result i32)))

;; instance of vtable
(global $Base.vtable___g_24 (ref $Base.vtable___type_26)
    ref.func $Base.foo___fun_62
    struct.new $Base.vtable___type_26)
(func $Base.foo___fun_62 (type $____type_53)
    (param $0_<this> (ref null $kotlin.Any___type_13)) (result i32)
    i32.const 1
    return)
```

- `$Base___type_36` has the usual vtable, itable, typeinfo, hashCode
- vtable contains a function reference to `Base.foo`.
  - (it's not a `data class`, so there's no `hashCode` or `equals` definitions or anything like that).

Derived:


```wasm
;; type definition
;; Same as Base___type_36 except super class is Base___type_36 and vtable is Derived.vtable
(type $Derived___type_42 (sub $Base___type_36 (struct
    (field (ref $Derived.vtable___type_39)) (field (ref null struct)) (field (mut i32)) (field (mut i32)))))
(type $Derived.vtable___type_39 (sub $Base.vtable___type_26 (struct
    (field (ref null $____type_53)))))
(type $____type_53 (func (param (ref null $kotlin.Any___type_13)) (result i32)))

;; vtable instance
(global $Derived.vtable___g_25 (ref $Derived.vtable___type_39)
    ref.func $Derived.foo___fun_64
    struct.new $Derived.vtable___type_39)
(func $Derived.foo___fun_64 (type $____type_53)
    (param $0_<this> (ref null $kotlin.Any___type_13)) (result i32)
    i32.const 2
    return)
```

Since `foo` is overridden, a function reference to `Derived.foo` is registered in the `Derived` vtable.


The `box` function looks like this.


```wasm
(func $box___fun_65 (type $____type_3)
    (local $0_d (ref null $Derived___type_42))
    ref.null none
    i32.const 1
    call $Derived.<init>___fun_63
    local.tee $0_d  ;; type: <root>.Derived
    call $bar___fun_66)
```

- `Drived.<init>` is the constructor as seen above.
- `$bar___fun_66` with Derived instance.


```wasm
(type $____type_92 (func (param (ref null $Base___type_36))))
(func $bar___fun_66 (type $____type_92)
    (param $0_f (ref null $Base___type_36)) (result i32)
    local.get $0_f  ;; type: <root>.Base
    local.get $0_f  ;; type: <root>.Base
    
    ;; virtual call: Base.foo
    struct.get $Base___type_36 0
    struct.get $Base.vtable___type_26 0
    call_ref (type $____type_53)
    
    return)
```

- First, push two references of type `Base___type_36` on the stack (in the code above we would pass a reference to an instance of `Derived`).
  - One to get the function reference to `foo` from `Base.vtable`.
  - The other is as a receiver given to `Base.foo`
- Get the vtable of `$0_f: (ref null $Base___type_36)` received as an argument in `struct.get $Base___type_36 0` (`Derived.vtable`)
- Get function reference to `foo` from vtable with `struct.get $Base.vtable___type_26 0` (`Derived.foo`)
- Function call with `call_ref`.


### interface dispatch

Kotlin (and Java...) has a single inheritance, but what if you are implementing multiple interfaces?



```kotlin
interface Base1 {
    fun foo(): Int
}
interface Base2 {
    fun bar(): Int
}
interface Base: Base1, Base2

class Derived: Base {
    override fun foo(): Int = 1
    override fun bar(): Int = 1
}

fun box() {
    val d = Derived()
    baz(d)
}
fun baz(b: Base) = b.foo() + b.bar()
```

Starts with `box___fun`


```wasm
(func $box___fun_64 (type $____type_3)
    (local $0_d (ref null $Derived___type_40))
    ref.null none
    call $Derived.<init>___fun_61
    local.tee $0_d  ;; type: <root>.Derived
    call $baz___fun_65
    drop)
```

Just `call $Derived.<init>` and call `call $baz___fun_65`. We will look at `Derived.<init>`.

```wasm
(type $Derived___type_40 (sub $kotlin.Any___type_16 (struct
    (field (ref $Derived.vtable___type_30))
    (field (ref null struct))
    (field (mut i32))
    (field (mut i32)))))

(func $Derived.<init>___fun_61 (type $____type_92)
    (param $0_<this> (ref null $Derived___type_40)) (result (ref null $Derived___type_40))
    
    ;; Object creation prefix
    local.get $0_<this>
    ref.is_null
    if
        
        ;; Any parameters
        global.get $Derived.vtable___g_24
        global.get $Derived.classITable___g_27
        i32.const 452
        i32.const 0
        
        struct.new $Derived___type_40
        local.set $0_<this>
    end
    
    local.get $0_<this>
    return)
```


We can see that `global.get Derived.classITable___g_27` sets the itable. What is this?

```wasm
;; instance of itable
(global $Derived.classITable___g_27 (ref $classITable___type_20)
    ref.func $Derived.foo___fun_62
    struct.new $Base1.itable___type_13
    ref.func $Derived.bar___fun_63
    struct.new $Base2.itable___type_14
    struct.new $Base.itable___type_15
    struct.new $classITable___type_20)

;; type definition
(type $classITable___type_20 (struct
    (field (ref null $Base1.itable___type_13))
    (field (ref null $Base2.itable___type_14))
    (field (ref null $Base.itable___type_15))))
(type $Base1.itable___type_13 (struct (field (ref null $____type_55))))
(type $Base2.itable___type_14 (struct (field (ref null $____type_55))))
(type $Base.itable___type_15 (struct))
(type $____type_55 (func (param (ref null $kotlin.Any___type_16)) (result i32)))
```

- Each `Derived` itable has a reference to an itable of `Base1`, `Base2` or `Base`.
- Each itable has a function reference of the function type declared in its interface.
  - In `Base` there is no function declaration, so an empty struct
- `global $Derived.classITable___g_27` registers a function reference to `Derived.foo` or `Derived.bar` in the itable.

Then we'll look at the implementation on the caller side (`baz`).


```wasm
(func $baz___fun_65 (type $____type_55)
    (param $0_b (ref null $kotlin.Any___type_16)) (result i32)
    local.get $0_b  ;; type: <root>.Base
    local.get $0_b  ;; type: <root>.Base
    
    ;; interface call: Base1.foo
    struct.get $kotlin.Any___type_16 1
    ref.cast $classITable___type_20
    struct.get $classITable___type_20 0
    struct.get $Base1.itable___type_13 0
    call_ref (type $____type_55)
    
    ;; ...
    
    i32.add
    return)
```

- First, two instances of `(param $0_b (ref null $kotlin.Any___type_16))` (actually instances of `Derived`) received as arguments are loaded onto the stack.
  - Similar to the virtual call example, one to get the function reference from itable and one receiver
- Get the itable of `$0_b` with `struct.get $kotlin.Any___type_16 1`.
- Cast this type to the `Derived` itable (`$classITable___type_20`)
  - Why, unlike vtable, do we downcast the type of itable at runtime from `(ref null struct)`?
- Get `Base1` itable with `struct.get $classITable___type_20 0`.
- `struct.get $Base1.itable___type_13 0` to get the function reference of type `Base1.foo`
- Call `Derived.foo` with `call_ref` with `$0_b` first on the stack as receiver
- Omit `Derived.bar` as it is the same.


### varargs
```kotlin
fun sum(vararg xs: Int): Int = xs.sum()
fun box() {
    sum(1, 2)
}
```

Maybe in the process of lowering to KotlinIR, varargs are converted to Arrays.


```wasm
(type $____type_54 (func (param (ref null $kotlin.IntArray___type_31)) (result i32)))
(func $sum___fun_65 (type $____type_54)
    (param $0_xs (ref null $kotlin.IntArray___type_31)) (result i32)
    local.get $0_xs  ;; type: kotlin.IntArray
    call $kotlin.collections.sum___fun_6
    return)
```

Definition of IntArray


```wasm
(type $kotlin.wasm.internal.WasmIntArray___type_15 (array (mut i32)))
(type $kotlin.IntArray___type_31 (sub $kotlin.Any___type_13 (struct
    (field (ref $kotlin.IntArray.vtable___type_21))
    (field (ref null struct))
    (field (mut i32))
    (field (mut i32))
    (field (mut (ref null $kotlin.wasm.internal.WasmIntArray___type_15))))))
```

caller side


```wasm
(func $box___fun_66 (type $____type_3)
    
    ;; Any parameters
    global.get $kotlin.IntArray.vtable___g_12
    ref.null struct
    i32.const 96
    i32.const 0
    
    i32.const 0
    i32.const 2
    array.new_data $kotlin.wasm.internal.WasmIntArray___type_15 1
    struct.new $kotlin.IntArray___type_31
    call $sum___fun_65
    drop)

;; dataidx = 1
(data "\01\00\00\00\02\00\00\00")
```

- `array.new_data $t $d: [i32, i32] -> [(ref $t)]`
  - `array.new_data $kotlin.wasm.internal.WasmIntArray___type_15 1` creates an array with RTT `$kotlin.wasm.internal.WasmIntArray___type_15` from the first data in data section from the first data in the data section.
  - operand is the offset and size in data.
  - The `i32.const 0` and `i32.const 2` are offset and size respectively.
- The operands `i32.const 96` `i32.const 0` from `global.get` are the arguments of `struct.new $kotlin.IntArray___type_31`.


### generic function

```kotlin
fun <T> id(x: T): T = x
fun box() {
    id(1)
}
```

```wasm
(func $box___fun_63 (type $____type_3)
    
    ;; vtable, itable, typeinfo, hashcode given to kotlin.Int___type_40
    ;; Any parameters
    global.get $kotlin.Int.vtable___g_13
    global.get $kotlin.Int.classITable___g_26
    i32.const 480
    i32.const 0
    
    i32.const 1
    struct.new $kotlin.Int___type_40  ;; box
    call $id___fun_62
    ref.cast $kotlin.Int___type_40
    struct.get $kotlin.Int___type_40 4  ;; name: value, type: kotlin.Int
    drop)

(func $id___fun_62 (type $____type_56)
    (param $0_x (ref null $kotlin.Any___type_13)) (result (ref null $kotlin.Any___type_13))
    local.get $0_x  ;; type: T of <root>.id
    return)

(type $____type_56 (func (param (ref null $kotlin.Any___type_13)) (result (ref null $kotlin.Any___type_13))))
```

- Boxing to `kotlin.Int` instead of `i32`
- `T` is the top type (here Any) satisfying the type constraints within kotlin types
- caller casts the result to the desired type


### generic class


```kotlin
class Box<T>(t: T) {
    var value = t
}
fun box() {
    val b = Box<Int>(1)
    b.value
}
```


The type definition of `Box` is. `Any` instead of `T`.


```wasm
(type $Box___type_38 (sub $kotlin.Any___type_13 (struct
    (field (ref $Box.vtable___type_27))
    (field (ref null struct))
    (field (mut i32))
    (field (mut i32))
    (field (mut (ref null $kotlin.Any___type_13)))))) ;; value
```


This is how the `Box` is instantiated and the fields accessed.

```wasm
(func $box___fun_63 (type $____type_3)
    (local $0_b (ref null $Box___type_38))
    ref.null none
    
    ;; Any parameters
    global.get $kotlin.Int.vtable___g_13
    global.get $kotlin.Int.classITable___g_27
    i32.const 512
    i32.const 0
    
    i32.const 1
    struct.new $kotlin.Int___type_42  ;; box
    call $Box.<init>___fun_62
    local.tee $0_b  ;; type: <root>.Box<kotlin.Int>
    struct.get $Box___type_38 4  ;; name: value, type: T of <root>.Box
    ref.cast $kotlin.Int___type_42
    struct.get $kotlin.Int___type_42 4  ;; name: value, type: kotlin.Int
    drop)
```

- Nothing special to mention until `call $Box.<init>___fun_62`. It's about boxing `Int`.
- Interesting is the part corresponding to `b.value`
  - Get the value with `struct.get $Box___type_38 4` (type `kotlin.Any`)
  - Cast the result with `ref.cast $kotlin.Int___type_42`
  - Finally, unboxing the Int.


### enum and pattern match


```kotlin
enum class Kind { A, B }
fun box() {
    bar(Kind.A)
}
fun bar(k: Kind) =
    when(k) {
        Kind.A -> 1
        Kind.B -> 2
    }
```

```wasm
(type $Kind___type_44 (sub $kotlin.Enum___type_32 (struct
    (field (ref $Kind.vtable___type_41)) ;; vtable
    (field (ref null struct)) ;; itable
    (field (mut i32)) ;; typeInfo
    (field (mut i32)) ;; hashCode
    (field (mut (ref null $kotlin.String___type_34))) ;; enum string representation
    (field (mut i32))))) ;; ordinal
```


Looking at the box function, the wasm expression for `Kind.A` is a call to something called `$Kind_A_getInstance___fun_80`.

```wasm
(func $box___fun_78 (type $____type_3)
    call $Kind_A_getInstance___fun_80
    call $bar___fun_79
    drop)
```

This is a function that calls the function `$Kind_initEntries___fun_76` and then returns an instance of `Kind.A` defined as global


```wasm
(func $Kind_A_getInstance___fun_80 (type $____type_103) (result (ref null $Kind___type_44))
    call $Kind_initEntries___fun_76
    global.get $Kind_A_instance___g_8  ;; type: <root>.Kind?
    return)
```

`$Kind_initEntries___fun_76`, as the name implies, creates instances of `Kind.A` and `Kind.B` and `global.set`.

```wasm
(func $Kind_initEntries___fun_76 (type $____type_3)
    ;; Initialisation checks to avoid multiple runs (omitted).
    ;; ...
    ref.null none
    
    ;; const string: "A"
    i32.const 29
    i32.const 1128
    i32.const 1
    call $kotlin.stringLiteral___fun_25
    
    i32.const 0 ;; A -> 0, B -> 1
    call $Kind.<init>___fun_77
    global.set $Kind_A_instance___g_8  ;; type: <root>.Kind?

    ;; Same for Kind.B
)
```

`$Kind.<init>___fun_77` is an object initialisation function similar to the ones we have seen so far.

Now we will look at the pattern matching part.


```wasm
(func $bar___fun_79 (type $____type_102)
    (param $0_k (ref null $Kind___type_44)) (result i32)
    (local $1_tmp0_subject (ref null $Kind___type_44))
    local.get $0_k  ;; type: <root>.Kind
    local.tee $1_tmp0_subject  ;; type: <root>.Kind
    call $Kind_A_getInstance___fun_80
    local.get $1_tmp0_subject  ;; type: <root>.Kind
    
    ;; virtual call: kotlin.Any.equals
    struct.get $kotlin.Any___type_13 0
    struct.get $kotlin.Any.vtable___type_12 0
    call_ref (type $____type_62)
    
    if (result i32)
        i32.const 1
    else
        local.get $1_tmp0_subject  ;; type: <root>.Kind
        call $Kind_B_getInstance___fun_81
        local.get $1_tmp0_subject  ;; type: <root>.Kind
        
        ;; virtual call: kotlin.Any.equals
        struct.get $kotlin.Any___type_13 0
        struct.get $kotlin.Any.vtable___type_12 0
        call_ref (type $____type_62)
        
        if (result i32)
            i32.const 2
        else
            call $kotlin.wasm.internal.throwNoBranchMatchedException___fun_30
            unreachable
        end
    end
    return)
```


It's long winded, but you can see that it is converted into an if-else like this.


```kotlin
if (k == Kind.A) { return 1; }
else {
    if (k == Kind.B) { return 2; }
    else { throw NoBranchMatchedException(...) }
}
```


Enum lowering is done around [here](https://github.com/JetBrains/kotlin/blob/a441a82357270f793dac3a378505c6c6993c44be/compiler/ir/backend.wasm/ src/org/jetbrains/kotlin/backend/wasm/WasmLoweringPhases.kt#L204-L267)


## Conculusion
- In Wasm generated by Rust(and C/C++), it was difficult to read the generated Wasm code because the allocation on linear memory and the pointer to those structures (`i32`) had no type.
  - In WasmGC, the engine takes care of the allocation by just `struct.new`, so the WASM code is much easier to read.
- From the compiler's point of view, the target language has become high-level, and implementation might be a bit easier (engine implementation seems to have become harder though).
- This time I observed only the ones that seem to be related to WasmGC, but I would like to investigate Wasm expressions in other high-level-constructs.
  - Exception handling
  - coroutine
  - threading
  - Optimised representation of string
  - unsigned xxx

## References
- [Introducing Kotlin/Wasm by Zalim Bashorov & Sébastien Deleuze @ Wasm I/O 2023 - YouTube](https://www.youtube.com/watch?v=LCtMC_IVCKo)
  - blog ver: [Introducing Kotlin/Wasm · seb.deleuze.fr](https://seb.deleuze.fr/introducing-kotlin-wasm/)
- [Kotlin Docs | Kotlin Documentation](https://kotlinlang.org/docs/home.html)
- [kotlinlang #webassembly](https://slack-chats.kotlinlang.org/c/webassembly)
- [Interface Dispatch | Lukas Atkinson](https://lukasatkinson.de/2018/interface-dispatch/)
- [How does dynamic dispatch work in WebAssembly?](https://fitzgeraldnick.com/2018/04/26/how-does-dynamic-dispatch-work-in-wasm.html)
  - How to implement dynamic dispatch in Rust. This one is implemented using `call_indirect`, but WasmGC introduces both struct and `typed function reference`, so it may be easier to use vtable or itable for each class(?).