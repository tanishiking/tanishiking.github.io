---
title: "Canonical ABI lift/lower Cheat Sheet"
publishDate: 2026-09-11
author: 'Rikito Taniguchi'
tags:
  - Wasm
---

The [Canonical ABI](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md) defines the mutual conversion between rich interface types defined in [WIT](https://component-model.bytecodealliance.org/design/wit.html) and their corresponding representations in Core Wasm (as well as function calling conventions between components).

Since the mapping between WIT and Core Wasm types/values is quite complex and easy to forget, I am creating a cheat sheet as a personal note. It might be useful as a supplementary reader to `CanonicalABI.md`.

I will ignore the `Future`, `Stream`, and `ErrorContext` types for now.

## Specialized value types

`tuple`, `flags`, `enum`, `options`, `result`, and `string` are defined as [specialized value types](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Explainer.md#specialized-value-types), which are special types represented by other types.

```
                    (tuple <valtype>*) ↦ (record (field "𝒊" <valtype>)*) for 𝒊=0,1,...
                    (flags "<label>"*) ↦ (record (field "<label>" bool)*)
                     (enum "<label>"+) ↦ (variant (case "<label>")+)
                    (option <valtype>) ↦ (variant (case "none") (case "some" <valtype>))
(result <valtype>? (error <valtype>)?) ↦ (variant (case "ok" <valtype>?) (case "error" <valtype>?))
                                string ↦ (list char)
```

`flags`/`string` are exceptions because they have different core wasm representations from `record`/`list` (so why were they defined as specialized types?), but other special types follow the same lift/lower behavior as `record` and `variant`.

## flattening

As described in the [flattening](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#flattening) section, the Canonical ABI basically defines data exchange between Components in terms of how data is stored/loaded on linear memory. However, when the number of function arguments or return values is small, for performance improvement, it is possible to exchange values through the stack like a normal core wasm function (flattening).

Therefore, a component function signature becomes the following signature in core wasm.

```python
MAX_FLAT_PARAMS = 16
MAX_FLAT_RESULTS = 1

def flatten_functype(opts, ft, context):
  flat_params = flatten_types(ft.param_types())
  flat_results = flatten_types(ft.result_types())
  if opts.sync:
    # If there are too many arguments, all arguments are passed via linear memory.
    # The stack instead contains the starting memory location where those arguments are stored.
    if len(flat_params) > MAX_FLAT_PARAMS:
      flat_params = ['i32']
    if len(flat_results) > MAX_FLAT_RESULTS:
      # Case where there are 2 or more results
      # Return values are returned through memory
      match context:
        case 'lift': # Case of exporting a function
          # The return value is placed in linear memory. The result stack has its memory offset.
          flat_results = ['i32']
        case 'lower': # Case of importing a function from another component
          # Add a pointer (offset in linear memory) to the parameters
          # The callee places the return value at this offset.
          # The caller loads the value from this offset after the function call.
          flat_params += ['i32']
          flat_results = []
    return CoreFuncType(flat_params, flat_results)
  else:
    # ... (case of async call, ignore for now)

def flatten_types(ts):
  # flatten_type is a function that converts a WIT type to a core wasm type
  return [ft for t in ts for ft in flatten_type(t)]
```

The above is the definition for type conversion, but for value conversion as well, switching between placing on the stack and storing/loading to/from linear memory is performed depending on the number of arguments/return values. For details, see [lifting and lowering values](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#lifting-and-lowering-values).

Here is a summary of the terminology.

- `lower_flat`: convert a component value to a core wasm value (place on the stack)
- `lift_flat`: convert a core wasm value to a component value (place on the stack)
- `lower_heap`/`store`: save a component value to linear memory
- `lift_heap`/`load`: read a core wasm value from linear memory and convert it to a component value

Below, I will summarize as simply as possible what kind of conversion is performed on WIT types with `lower/lift flat/heap`.

## bool

- `lower_flat`: 1 if True, 0 if False
- `lift_flat`: False if 0, True if 1 or more
  - core type: `i32`
- `store`/`load`: convert to 0 or 1 and save in linear memory as 1 byte

## u8/u16/u32/u64

- `lower_flat`: no value conversion
- `lift_flat`: ignore the upper bits of the core value (i32 or i64)
  - For example, when lifting a 32-bit core value to a u8 component-level value, the upper 24 bits are ignored.
  - core type: `i32` (u64 is `i64`)
- `load`/`store`: save in linear memory in little endian. 1 byte to 4 bytes for u8 to u64 respectively.

## s8/s16/s32/i64

- `lower_flat`: interpret the byte string represented in 2's complement as an unsigned 32-bit integer
- `lift_flat`: similar to u8-u64, ignore the upper bits of the core value, then, if the value is greater than the upper limit of the signed range of the target type, subtract the width of the target type from the value. This ensures that the value is within the valid signed range of the target type.
  - core type: `i32` (i64 is `i64`)
- `load`/`store`: save in linear memory in little endian. 1 byte to 4 bytes for s8 to s64 respectively.

```python
# https://github.com/WebAssembly/component-model/blob/ee4822aacbce083599f692fc8c8efb08db8d3f3a/design/mvp/canonical-abi/definitions.py#L1572-L1578
def lift_flat_signed(vi, core_width, t_width):
  i = vi.next('i' + str(core_width))
  assert(0 <= i < (1 << core_width))
  i %= (1 << t_width)
  if i >= (1 << (t_width - 1)):
    return i - (1 << t_width)
  return i
```

## f32/f64

Encoded by a function called `maybe_scramble_nan32(f)`.

If the floating point number is NaN, it generates a deterministic NaN representation value (depending on ComponentModel options) or a random i32 value and converts it to f32 with `f32.reinterpret_i32`. Otherwise, there is no value conversion.

```python
def maybe_scramble_nan32(f):
  if math.isnan(f):
    if DETERMINISTIC_PROFILE:
      f = core_f32_reinterpret_i32(CANONICAL_FLOAT32_NAN)
    else:
      f = core_f32_reinterpret_i32(random_nan_bits(32, 8))
    assert(math.isnan(f))
  return f
```

When lifting a float value, it checks if it is NaN, and if so, it converts the integer value corresponding to the specified NaN to a component value with `f32.reinterpret_i32`.

```python
DETERMINISTIC_PROFILE = False # or True
CANONICAL_FLOAT32_NAN = 0x7fc00000
CANONICAL_FLOAT64_NAN = 0x7ff8000000000000

def canonicalize_nan32(f):
  if math.isnan(f):
    f = core_f32_reinterpret_i32(CANONICAL_FLOAT32_NAN)
    assert(math.isnan(f))
  return f
```

I haven't delved into the representation of NaN.

- `lower_flat`: `maybe_scramble_nan32(f)`
- `lift_flat`: `canonicalize_nan32(f)`
- `store`: store the integer value interpreted as i32 with `i32.reinterpret_f32` of the value from `maybe_scramble_nan32(f)`
- `load`: load the integer value, then `canonicalize_nan32` after `f32.reinterpret_i32`

## char

`char` is assumed to be a [Unicode Scalar Value](https://unicode.org/glossary/#unicode_scalar_value), and there is no value conversion (`i32`).

Therefore, both `lower` and `lift` are the same as for u32.

## string
A component-level string is actually represented by the following triplet.

- `src`: actual string data (in the example, a decoded python str is used, but is it VM implementation dependent?)
- `src_encoding`: a string indicating the encoding of the string (in core wasm) ("utf8", "utf16", "latin1+utf16")
- `src_tagged_code_units`: the number of encoded code units of the string in `src_encoding`

There are three `src_encoding`s: `utf8`, `utf16`, and `latin1+utf16`. In the case of `latin1+utf16`, if the most significant bit of `src_tagged_code_units` is 1, it's `utf16`, and if it's 0, it's `latin1`, so it seems that two character codes can be used within a component.

A string in core wasm is encoded (with `src_encoding`) and placed in linear memory, and is represented by a pair of the offset in linear memory and the number of code units (`[i32, i32]`).

> Since strings and variable-length lists are stored in linear memory, lifting can reuse the previous definitions; only the resulting pointers are returned differently (as i32 values instead of as a pair in linear memory). Fixed-length lists are lowered the same way as tuples
> https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#flat-lowering

- `lower_flat`
  - Encodes `src` with the encoding specified by the destination component (note that this is different from `src_encoding`), stores it in linear memory, and returns its offset and length.
  - At this time, memory is allocated using [realloc](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Explainer.md#canonical-abi) (probably provided by each Component), and `src_encoding` and `src_tagged_code_units` serve as hints for allocating memory.
- `lift_flat`
  - Loads a byte string from linear memory based on the pair (`[i32, i32]`).
  - Decodes the byte string into a string according to the encoding specified by the source component.
  - The triplet of the decoded string, encoding, and number of code units becomes the component-level string.
- `load`/`store`
  - Similar to `lower/lift_flat`, store/load data to/from linear memory (allocated with realloc).
  - Store/load its offset and number of code units to/from linear memory.

> Storing strings is complicated by the goal of attempting to optimize the different transcoding cases. In particular, **one challenge is choosing the linear memory allocation size before examining the contents of the string.** The reason for this constraint is that, in some settings where single-pass iterators are involved (host calls and post-MVP adapter functions), examining the contents of a string more than once would require making an engine-internal temporary copy of the whole string, which the component model specifically aims not to do. **To avoid multiple passes, the canonical ABI instead uses a realloc approach to update the allocation size during the single copy.** A blind realloc approach would normally suffer from multiple reallocations per string (e.g., using the standard doubling-growth strategy). However, as already shown in load_string above, **string values come with two useful hints: their original encoding and byte length. From this hint data, store_string can do a much better job minimizing the number of reallocations.**
> https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#storing

For details, see [store_string](https://github.com/WebAssembly/component-model/blob/ee4822aacbce083599f692fc8c8efb08db8d3f3a/design/mvp/CanonicalABI.md#storing) and [load_string](https://github.com/WebAssembly/component-model/blob/ee4822aacbce083599f692fc8c8efb08db8d3f3a/design/mvp/CanonicalABI.md#loading).

## list

The list type is parameterized by the following two elements:

- `t`: element type
- `l`: length of the list (`None` for variable-length lists)

```python
@dataclass
class ListType(ValType):
  t: ValType
  l: Optional[int] = None
```

- For fixed-length lists:
  - A sequence of lowered values of each element.
  - For example, `ListType(i32, 3)` becomes the type `[i32, i32, i32]`.
  - And `ListType(string, 2)` becomes `[i32, i32, i32, i32]`.
- For variable-length lists:
  - Similar to strings, values are stored in linear memory (the contents of the list are placed in order from the front).
  - The pair of the starting offset and length (`[i32, i32]`) becomes the core wasm value.

Thus, the representation in core wasm differs between fixed-length and variable-length lists.

- `lower_flat`
  - Just convert according to the rules above.
- `lift_flat`
  - If the target type is a fixed-length list, just lift each core value.
  - If it's a variable-length list, load and lift values by `elem_size` of the element type from the range of linear memory represented by the pair.
    - `elem_size` is the length of the component-level type on the variable-length list.
- `load`/`store`
  - Fixed-length: What was placed on the stack with `flat` is simply placed in linear memory.
  - Variable-length: Similar to strings, data is placed in a range (allocated with realloc), and the pair of its offset and length is placed in linear memory.

## record

The record type is parameterized by a list of `FieldType`s, and `FieldType` has a label name and an element type.

```python
@dataclass
class FieldType:
  label: str
  t: ValType

@dataclass
class RecordType(ValType):
  fields: list[FieldType]
```

**The component-level value is an associative array of label-value pairs.**

### lower_flat/lift_flat

- `lower_flat`:
  - Just `lower_flat` the values of the fields in order.
  - For example, `{"a": u32, "b": string}` is lowered to `[i32, i32, i32]`. The field names are not passed to core wasm.
- `lift_flat`:
  - Just `lift_flat` the fields in order with the element type and label them.
  - To lift `[i32, i32, i32]` to `{"a": u32, "b": string}`, first lift the first i32 as u32 and assign it to the value of "a", then lift `[i32, i32]` as a string... and so on.

### load/store

Placing on memory is a bit more difficult, and requires talking about alignment and elem_size. This is because **for each element of a record, we want to place an N-byte element at an N-byte alignment.**

Storing a record in linear memory can be simply expressed by the following function:

```python
# v - associative array representing the record
# ptr - starting position in linear memory to store the record
# fields - fields of the record
def store_record(cx, v, ptr, fields):
  # For each field
  for f in fields:
    # alignment - a function that takes a WIT type and returns its alignment
    # align_to - a function that takes a ptr and alignment and pads the ptr to reach that alignment boundary
    # First, pad the ptr to the alignment boundary of the field's type and move the ptr
    ptr = align_to(ptr, alignment(f.t))
    # Once the ptr is moved to the alignment boundary, save the value there
    store(cx, v[f.label], f.t, ptr)
    # elem_size - a function that takes a WIT type and returns its byte size
    # Move the ptr by the amount of the saved value
    ptr += elem_size(f.t)
```

For example, for a record like `(record (field "a" u32) (field "b" u8) (field "c" u16) (field "d" u8))`:

- u32: alignment 4, element size 4
- u8: alignment 1, element size 1
- u16: alignment 2, element size 2
- entire record: alignment 4 (same as the alignment of the largest field), element size 12 (described later)

(For the alignment and element size of each, see [alignment](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#alignment) and [elem_size](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#element-size). The elem_size of a record might be easier to understand after reading the explanation below.)

- a (s = 0)
  - `align_to(0, 4) = 0`, so the offset is already at the alignment.
  - `s += 4`
- b (s = 4)
  - `align_to(4, 1) = 4`
  - `s += 1`
- c (s = 5)
  - `align_to(5, 2) = 6`. A 2-byte element should be placed at a 2-byte alignment, so `align_to` aligns it to the 6th byte.
  - `s += 2`
- d (s = 8)
  - `align_to(8, 1) = 8`
  - `s += 1`

Therefore, it should be arranged as follows in the end:

| byte  | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|-------|---|---|---|---|---|---|---|---|---|
| field | a | a | a | a | b | - | c | c | d |

And the `elem_size` of the record looks like 9, but actually, at the end, `s` is aligned to the alignment of the entire record. This ensures that the data structures following this record are correctly aligned.

`align_to(9, 4) = 12`. This makes the total size of the record 12.

To read a record from linear memory, you can just do the reverse operation.

```python
def load_record(cx, ptr, fields):
  record = {}
  for field in fields:
    ptr = align_to(ptr, alignment(field.t))
    record[field.label] = load(cx, ptr, field.t)
    ptr += elem_size(field.t)
  return record
```

## variant

```python
@dataclass
class CaseType:
  label: str
  t: Optional[ValType]

@dataclass
class VariantType(ValType):
  cases: list[CaseType]
```

For example, for a variant type like `(variant (case "a" u32) (case "b" string))`:

- The component-level value is a dictionary with only one element, like `{"a": 42}` or `{"b": "foo"}` (the key is the case label of the variant type, and the value is a value corresponding to the type of that case).
- The core value is `[<case_index as i32>, <lower values>...]`.
  - At the beginning is an `i32`: the case_index. For example, for `{"a": 42}`, it matches the 0th case, so it's 0. For `{"b": "foo"}`, it matches the 1st case, so it's 1.
  - After that comes the lowered value of the value. For example, 42 (`i32`) is also i32 in core wasm. A string is represented by `[i32, i32]`.
  - So, for example, `[0, 42]` (matches the 0th case, and the value is i32 type 42) or `[1, 100, 3]` (matches the 1st case, and the value is a string of length 3 stored at offset 100).

### Zero-padding
We know how the value is converted in the core value, but what about the type? We want a 1:1 correspondence between the component-level type and the core type. It's not good if the type is different for `[0, 42]` and `[1, 100, 3]` depending on which case of the variant it is.

What to do is to match the longest value type. In this case, the case `(case "b" string)` has the longest type `[i32, i32, i32]`, so the core type for `(variant (case "a" u32) (case "b" string))` is `[i32, i32, i32]`. Cases with insufficient length, like `(case "a" u32)`, are zero-padded at the end to match the type (becoming `[0, 42, 0]`).

### approximation
It's fine if the value types match to some extent like in the case above, but what if the core wasm representations of the values differ not only in length but also in type, like `(variant (case "a" f64) (case "b" string))`? When `a: [f64]` vs `b: [i32, i32]`, `f64` and `i32` have a type mismatch. In such cases, select the tightest type that can represent both. Specifically:

```python
def join(a, b):
  if a == b: return a
  if (a == 'i32' and b == 'f32') or (a == 'f32' and b == 'i32'): return 'i32'
  return 'i64'
```

- `i32` vs `f32` -> `i32` (can be mutually converted with `f32.reinterpret_i32`, etc.)
- In other cases, `i64`
  - `f64` vs `i64` can be mutually converted with `f64.reinterpret_i64`, etc.
  - Any other case can also be represented by `i64`.

Therefore, the core wasm type of `(variant (case "a" f64) (case "b" string))` is `[i32, i64, i32]`.

> Variant flattening is more involved due to the fact that each case payload can have a totally different flattening. Rather than giving up when there is a type mismatch, **the Canonical ABI relies on the fact that the 4 core value types can be easily bit-cast between each other** and defines a join operator to pick the tightest approximation. What this means is that, regardless of the dynamic case, all flattened variants are passed with the same static set of core types, which may involve, e.g., reinterpreting an f32 as an i32 or zero-extending an i32 into an i64.
> https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#flattening

Isn't this impossible for reference types?

### lower_flat/lift_flat

- `lower_flat`
  - Get the case index of the variant that matches the value.
  - `lower_flat` the value. If there is a type mismatch with the core wasm type of the variant, adjust the type with approximation.
    - `i32.reinterpret` an f32 value, etc.
  - Finally, zero-pad to match the length of the variant's type.
- `lift_flat`
  - The first value of the core wasm value is the index of the matching case, and get the corresponding `label`.
  - In a variant, `f32` might be stored as `i32`, so decode it to f32, etc. After reading everything, `lift_flat` it and make it the `value`.
  - The zero-padded part is discarded.
  - `{label: value}`

### load/store

Basically, you just convert the value converted with `lower_flat` to linear memory as it is, but there is a slight difference.

`discriminant_type(cases)` calculates the minimum size required to hold the case_index at the beginning, depending on the number of cases. If it's 256 or less, 1 byte is sufficient, so u8. If it's 257-65536, 16 bytes are sufficient, so u16.

```python
def discriminant_type(cases):
  n = len(cases)
  assert(0 < n < (1 << 32))
  match math.ceil(math.log2(n)/8):
    case 0: return U8Type()
    case 1: return U8Type()
    case 2: return U16Type()
    case 3: return U32Type()
```

When saving a variant to linear memory:

- First, save the index of the matching case in the range of the type calculated by `discriminant_type`.
- Move the ptr to the alignment boundary of the longest case among the variant's cases.
  - This is because the payload of a variant needs a size that matches the longest of all cases.

```python
def store_variant(cx, v, ptr, cases):
  case_index, case_value = match_case(v, cases)
  disc_size = elem_size(discriminant_type(cases))
  store_int(cx, case_index, ptr, disc_size)
  ptr += disc_size
  ptr = align_to(ptr, max_case_alignment(cases))
  c = cases[case_index]
  if c.t is not None:
    store(cx, case_value, c.t, ptr)

def match_case(v, cases):
  [label] = v.keys()
  [index] = [i for i,c in enumerate(cases) if c.label == label]
  [value] = v.values()
  return (index, value)
```

`load_variant` just does the reverse of this operation.

## flags

For example, for a flag type `(flags "a" "b" "c")`:

- component-level value: `{"a": False, "b": False, "c": True}`, etc.
- core wasm value: `0b100` (c is True, a and b are False), a bitvector.
  - From the beginning of the label (a), place 0 or 1 in order from the **least** significant bit.

`lift_flat`/`lower_flat` is obvious from this correspondence. `store`/`load` also just converts the bitvector to an integer value and loads/stores it to/from linear memory.

## resource

To understand what a resource is, it seems easy to look at something like [WIT and Rust: Resources](https://zenn.dev/chikoski/articles/wit-and-rust-resource).

- [resource.new](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#canon-resourcenew) takes a resource representation (currently only i32 is available) and returns an index (i32) pointing to the ResourceHandle registered in the Component.
  - Does the resource rep contain the starting position of the memory where the resource entity is stored?
- [resource.rep](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#canon-resourcerep)
  - When you want to use the resource representation directly, it takes a resource handle index and returns the resource rep.
  - I don't really understand when I would want to use this.
- [resource.drop](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#canon-resourcedrop)
  - Takes a resource handle index and removes the Resource Handle from the Component.
  - It also executes the destructor if the resource has ownership.
  - Is it like freeing the memory area that held the resource entity in the destructor? I wonder how to handle reference types.

To be honest, I don't really understand how to use these canonical instructions... But for now, it's probably enough to know that when using a resource created by another component in our core module, a resource handle index (i32) is passed. (It will be a problem when we want to make our wasm module export resources).

> In the Component Model, handles are lifted-from and lowered-into i32 values that index an encapsulated per-component-instance handle table that is maintained by the canonical function definitions described below. In the future, handles could be backwards-compatibly lifted and lowered from [reference types] (via the addition of a new canonopt, as introduced below).

So, a resource handle is lowered to the index of the ResourceTable in the Component that points to that resource (this is why they call it something like a file descriptor).

`lower_borrow` and `lower_own` each add a new `ResourceHandle` element to the handle table of the current component instance for a borrow/own handle, and return `i32` (resource handle index) as the core wasm representation.
From the VM's perspective, the behavior when lowering/lifting a borrowed resource and an owned resource is different, but from the compiler's perspective, that difference is not visible and it is just lowered to `i32`.

- `lift_borrow` and `lift_own` each take a Resource Handle index (i32) and return `resource.rep`. In the process, they may or may not remove the resource from the ResourceHandle Table. I haven't looked into this area properly.
- `load`/`store` just makes `(lift|lower)_(borrow|own)` a resource handle and loads/stores that i32 to/from linear memory.

## Example

What would be the core wasm type when importing the [blocking-write-and-flush function of the output-stream resource](https://github.com/WebAssembly/WASI/blob/5120d70557cc499b98ee21b19f6066c29f1400a4/wasip2/io/streams.wit#L154-L181)?

```wit
resource output-stream {
  blocking-write-and-flush: func(
      contents: list<u8>
  ) -> result<_, stream-error>;
}
```

The answer is `(func (param i32 i32 i32 i32))`. Why?

- The first three i32s are for the resource (i32) and the variable-length list (i32, i32) respectively.
- The last i32 is a pointer for passing the result.
  - When a result type has multiple core wasm types, data is exchanged via linear memory, and an i32 pointer is placed in the parameters.
  - `result<_, stream-error>` is `(variant (case "ok") (case "error" stream-error))`. `stream-error` is also a variant type, and it is clear that it will be represented by two or more i32s. (I'll omit how it's placed in linear memory...)
