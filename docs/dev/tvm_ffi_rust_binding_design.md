<!--- Licensed to the Apache Software Foundation (ASF) under one -->
<!--- or more contributor license agreements.  See the NOTICE file -->
<!--- distributed with this work for additional information -->
<!--- regarding copyright ownership.  The ASF licenses this file -->
<!--- to you under the Apache License, Version 2.0 (the -->
<!--- "License"); you may not use this file except in compliance -->
<!--- with the License.  You may obtain a copy of the License at -->
<!--- -->
<!---   http://www.apache.org/licenses/LICENSE-2.0 -->
<!--- -->
<!--- Unless required by applicable law or agreed to in writing, -->
<!--- software distributed under the License is distributed on an -->
<!--- "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY -->
<!--- KIND, either express or implied.  See the License for the -->
<!--- specific language governing permissions and limitations -->
<!--- under the License. -->

# TVM-FFI Rust Binding Design Notes

This document captures the current architecture of the experimental Rust binding for TVM-FFI,
including runtime crates and the `stubgen` code generation toolchain.

## 1. Scope and reading map

Workspace root:

- `rust/README.md`

Core crates:

- `rust/tvm-ffi-sys` (raw C ABI):
  - `src/c_api.rs`
  - `src/c_env_api.rs`
  - `build.rs`
- `rust/tvm-ffi` (safe/high-level binding):
  - `src/lib.rs`
  - `src/object.rs`
  - `src/object_wrapper.rs`
  - `src/any.rs`
  - `src/type_traits.rs`
  - `src/function.rs`
  - `src/function_internal.rs`
  - `src/extra/module.rs`
  - `src/macros.rs`
  - `build.rs`
- `rust/tvm-ffi-macros` (proc macros):
  - `src/lib.rs`
  - `src/object_macros.rs`
  - `src/utils.rs`
- `rust/tvm-ffi-stubgen` (generator):
  - `src/main.rs`
  - `src/cli.rs`
  - `src/lib.rs`
  - `src/ffi.rs`
  - `src/model.rs`
  - `src/schema.rs`
  - `src/generate.rs`
  - `src/utils.rs`

Validation sources:

- `rust/tvm-ffi/tests/*.rs`
- `rust/tvm-ffi-stubgen/tests/stubgen.rs`

## 2. Layered architecture

Rust side is intentionally split by safety boundaries:

1. **FFI boundary (`tvm-ffi-sys`)**
   - `unsafe extern "C"` declarations that mirror C ABI types/functions.
   - no ownership ergonomics; almost direct ABI mapping.
2. **Safe runtime (`tvm-ffi`)**
   - Rust-native ownership wrappers (`ObjectArc`, `Function`, `Any`, etc.);
   - typed conversions and error mapping;
   - runtime dynamic library/module access.
3. **Codegen support (`tvm-ffi-stubgen`)**
   - loads TVM-FFI shared library;
   - extracts reflection metadata;
   - emits Rust wrappers with typed API surface.
4. **Macro ergonomics (`tvm-ffi-macros`, macro_rules! in `tvm-ffi`)**
   - derive and helper macros to reduce boilerplate.

## 3. `tvm-ffi-sys`: raw ABI layer

### 3.1 Responsibilities

- Define ABI-compatible structs/unions/enums.
- Expose all imported C symbols as Rust extern functions.
- Keep signatures and constants synchronized with `include/tvm/ffi/c_api.h`.

### 3.2 Build/link integration

`build.rs` is responsible for finding/linking `libtvm_ffi` (via environment and helper tooling),
so downstream crates can compile and run against the C++ runtime.

This crate should avoid policy logic; it is a mechanically faithful ABI bridge.

## 4. `tvm-ffi`: safe runtime binding

### 4.1 Object ownership and identity

Key types:

- `object::Object`: Rust representation of object header-compatible payload.
- `object::ObjectArc<T>`: intrusive strong reference wrapper analogous to C++ `ObjectPtr`.
- trait set:
  - `ObjectCore`
  - `ObjectCoreWithExtraItems`
  - `ObjectRefCore`

Core behavior:

- clone/drop on `ObjectArc<T>` call C refcount APIs (`IncRef`/`DecRef`);
- dynamic type checks and casts rely on TVM-FFI runtime type metadata;
- wrapper types can model typed object refs while preserving opaque ABI handles.

### 4.2 Type-erased transport: `AnyView` and `Any`

`any.rs` mirrors C++ transport model:

- `AnyView`: borrowed ABI value view (non-owning).
- `Any`: owned ABI value with drop semantics for heap/object payloads.

`TryFrom`, `From`, and helper conversions support primitives, options, object wrappers,
and other TVM-FFI-compatible types.

### 4.3 Conversion traits: `AnyCompatible`

`type_traits.rs` defines `AnyCompatible` as the central conversion contract:

- conversion from Rust type -> packed ABI argument;
- strict and cast-style conversion from packed ABI -> Rust type;
- ownership-sensitive behavior for object/reference payloads.

This trait is the Rust analogue of C++ `TypeTraits<T>`.

### 4.4 Function wrapper model

`function::Function` is the runtime callable handle:

- invoke:
  - `call_packed`
  - `call_tuple`
  - `call_tuple_with_len`
- lookup/register globals:
  - `Function::get_global`
  - `Function::register_global`
- construction from Rust callables:
  - `Function::from_packed`
  - `Function::from_typed`
  - `Function::from_extern_c`

Internal adapters in `function_internal.rs` (`AsPackedCallable`, tuple packing traits/macros)
bridge typed Rust closures into packed ABI callbacks.

### 4.5 Error model

Rust API returns `Result<T, Error>` and maps C ABI failures into rich Rust errors:

- C function returns status code.
- On error, binding reads raised error object from runtime TLS.
- Rust error object captures code + message + context helpers.

Macros like `check_safe_call!`, `bail!`, `ensure!`, `attach_context!` reduce repetitive
error handling patterns in wrapper implementations.

### 4.6 Module loading and symbol access

`extra::module::Module` wraps runtime module object behavior:

- `Module::load_from_file(path)` -> call into C++ module loader globals.
- `Module::get_function(name)` -> fetch exported `Function`.

This allows Rust applications to consume TVM-FFI-exported shared libraries with the same
runtime semantics used by Python/C++ callers.

## 5. `tvm-ffi-macros`: derive support

Procedural derives are focused on object boilerplate:

- `derive(Object)`
- `derive(ObjectRef)`

They generate trait impls connecting user Rust structs/wrappers to object core traits and
runtime type metadata hooks. This reduces handwritten unsafe glue code and encourages
uniform wrapper style.

## 6. `tvm-ffi-stubgen`: metadata-driven Rust wrapper generation

### 6.1 Goal

Given a TVM-FFI shared library, generate a Rust crate (or module set) with:

- object wrappers for reflected types,
- typed method/field accessors,
- typed global function wrappers,
- crate scaffolding (`Cargo.toml`, `lib.rs`, module files).

### 6.2 Pipeline overview

1. **Load runtime and target shared object**
   - via `libloading` and runtime entrypoints in `ffi.rs`.
2. **Query reflection metadata**
   - enumerate global function names;
   - enumerate/resolve type keys and metadata payloads.
3. **Parse schema**
   - `schema.rs` converts metadata (often JSON-like payloads) into normalized forms.
4. **Build internal model**
   - `model.rs` stores functions/types/fields/method signatures for generation.
5. **Render code**
   - `generate.rs` emits Rust source artifacts and crate structure.
6. **Write outputs**
   - `utils.rs` handles directory/file creation and formatting steps.

`tests/stubgen.rs` validates generated outputs against a test shared library.

### 6.3 Generated wrapper shape (conceptual)

For an object type:

- define wrapper struct implementing `ObjectWrapper`/object ref traits;
- expose reflected fields via getter/setter methods;
- expose reflected methods as typed Rust methods that delegate to underlying `Function`.

For a global function:

- generate typed wrapper function;
- lazily resolve runtime `Function`;
- convert Rust args via `AnyCompatible`, invoke packed call, and convert return.

### 6.4 Why codegen is needed

Without stub generation, consuming reflected APIs from Rust would require:

- string-based function/type lookups,
- manual type conversions,
- repetitive boilerplate and error-prone downcasts.

`stubgen` turns reflection metadata into static Rust APIs that are easier to discover,
autocomplete, and type-check.

## 7. End-to-end call path (C++ -> Rust consumer)

1. C++ library registers classes/functions via TVM-FFI reflection and global registry.
2. Shared library exports become discoverable at runtime.
3. `tvm-ffi-stubgen` loads the library and emits Rust wrappers.
4. Rust application links generated crate + `tvm-ffi`.
5. Runtime calls flow through:
   - generated typed wrapper ->
   - `tvm_ffi::Function` packed invocation ->
   - C ABI ->
   - C++ `FunctionObj` / reflected method.
6. Return value/error is translated back through `Any` + Rust `Result`.

## 8. Design constraints and current limitations

1. **Experimental status**
   - API shapes and macro surfaces may still evolve.
2. **Runtime dependency**
   - execution depends on locating compatible `libtvm_ffi`.
3. **Metadata fidelity**
   - `stubgen` quality depends on completeness/consistency of reflection metadata.
4. **Type coverage**
   - advanced/custom types may require manual trait implementations or generator updates.
5. **Unsafe boundary remains central**
   - safe wrappers reduce but cannot remove all ABI-related risks.

## 9. Practical extension checklist

When extending Rust runtime crate:

1. Add/adjust `tvm-ffi-sys` ABI declarations first.
2. Implement safe wrapper with explicit ownership/error behavior.
3. Add tests under `rust/tvm-ffi/tests`.
4. Validate interoperability with a real or test shared library.

When extending `stubgen`:

1. Update `model.rs` to represent new metadata concept.
2. Update `schema.rs` parser and normalization.
3. Update `generate.rs` templates/rendering logic.
4. Add integration test asserting generated API signatures.

## 10. Relationship to C++ native design

Rust binding is not a separate runtime; it is a **typed facade** over the same C++ runtime:

- same object header/refcount/type registry semantics;
- same packed function call ABI;
- same global function/module discovery;
- same reflection metadata as source of truth.

Therefore, any change in C++ registration metadata or ABI semantics should be reviewed for:

- `tvm-ffi-sys` compatibility,
- `tvm-ffi` conversion/ownership assumptions,
- `stubgen` schema parsing and template output.
