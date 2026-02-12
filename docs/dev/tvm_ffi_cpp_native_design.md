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

# TVM-FFI C++ Native Design Notes

This document is an implementation-oriented design memo for the C++ side of TVM-FFI.
It focuses on how runtime objects, references, registration, and dynamic dispatch are
implemented in the current codebase.

## 1. Scope and reading map

Primary files:

- ABI and C APIs: `include/tvm/ffi/c_api.h`
- Object model: `include/tvm/ffi/object.h`, `src/ffi/object.cc`
- Function model: `include/tvm/ffi/function.h`, `src/ffi/function.cc`
- Reflection and registration:
  - `include/tvm/ffi/reflection/registry.h`
  - `include/tvm/ffi/reflection/accessor.h`
  - `include/tvm/ffi/reflection/creator.h`
  - `include/tvm/ffi/reflection/overload.h`
- Memory helpers: `include/tvm/ffi/memory.h`
- Type-erased value transport: `include/tvm/ffi/any.h`, `include/tvm/ffi/type_traits.h`
- Module loading:
  - `include/tvm/ffi/extra/module.h`
  - `src/ffi/extra/module.cc`
  - `src/ffi/extra/library_module.cc`
  - `src/ffi/extra/library_module_dynamic_lib.cc`
- Error propagation and init: `src/ffi/init_once.cc`, error APIs in `c_api.h`

Companion tests:

- `tests/cpp/test_object.cc`
- `tests/cpp/test_reflection.cc`
- `tests/cpp/test_function.cc`
- `tests/cpp/test_overload.cc`
- `tests/cpp/testing_object.h`

## 2. Runtime architecture (high-level)

TVM-FFI C++ runtime can be viewed as four cooperating subsystems:

1. **Object/type subsystem**
   - intrusive reference-counted heap objects (`Object`, `ObjectPtr`, `ObjectRef`);
   - runtime type table keyed by `type_index` and `type_key`.
2. **Function subsystem**
   - runtime `FunctionObj` wrapper with dual call interfaces (`cpp_call`, `safe_call`);
   - global function registry and cross-language packed call convention.
3. **Reflection/registration subsystem**
   - metadata-driven registration for object constructors, fields, methods, and globals;
   - optional overload dispatch and type attributes.
4. **Module/dynamic loading subsystem**
   - load shared libraries as modules, resolve symbols, and expose exported functions.

The stable boundary between languages is the C ABI (`TVMFFI*` structs + `extern "C"` APIs).

## 3. Object and reference model

### 3.1 `TVMFFIObject` header and ownership bits

Every heap object managed by TVM-FFI starts with a stable C header (`TVMFFIObject`):

- `combined_ref_count` (`uint64_t`)
  - lower 32 bits: strong count
  - upper 32 bits: weak count
- `type_index`: runtime type id
- `deleter`: function pointer for destruction/free policy

This is **intrusive** reference counting: counts live inside the object itself, not in an
external control block.

### 3.2 C++ wrappers and their roles

- `tvm::ffi::Object`
  - base class for heap-allocated FFI objects.
- `tvm::ffi::ObjectPtr<T>`
  - strong smart pointer (increment/decrement strong count).
- `tvm::ffi::WeakObjectPtr<T>`
  - weak pointer (tracks liveness, does not keep object alive).
- `tvm::ffi::ObjectRef`
  - lightweight managed handle used by high-level APIs and language bindings.

In practice, user-visible object references should be modeled as subclasses of `ObjectRef`.

### 3.3 Lifecycle and deletion

Ownership transitions are handled by C APIs (`TVMFFIObjectIncRef`, `TVMFFIObjectDecRef`)
and C++ wrappers.

Destruction has two phases:

1. strong count drops to zero -> object destructor executes;
2. weak count also reaches zero -> memory gets released.

The `deleter` callback interprets flag bits (`TVMFFIObjectDeleterFlagBitMask`) to execute
the correct phase.

### 3.4 Allocation helpers

`include/tvm/ffi/memory.h` provides:

- `make_object<T>(...)`: allocate and construct an object.
- `make_inplace_array_object<T>(...)`: contiguous allocation for object + inline array payload.

Allocation goes through aligned alloc helpers to preserve ABI/layout guarantees.

### 3.5 Type identity and subtype checks

TVM-FFI does not rely on C++ RTTI (`dynamic_cast` / `typeid`) for runtime checks.

Type identity is runtime-managed by:

- `type_index` (fast integer id on each object),
- `type_key` (string key like `"my_ext.MyType"`),
- parent/child topology tracked by global type table (`src/ffi/object.cc`).

`Object::IsInstance<T>()`, `ObjectRef::as<T>()`, and `Any::cast<T>()` all depend on this
table rather than C++ RTTI.

## 4. User class declaration and registration

### 4.1 Declaring object classes

User classes inherit from `Object` (or a derived object type) and declare metadata via:

- `TVM_FFI_DECLARE_OBJECT_INFO(...)`
- `TVM_FFI_DECLARE_OBJECT_INFO_FINAL(...)`

Reference classes inherit from `ObjectRef` and define wrappers via:

- `TVM_FFI_DEFINE_OBJECT_REF_METHODS_NULLABLE(...)`
- `TVM_FFI_DEFINE_OBJECT_REF_METHODS_NOTNULLABLE(...)`

`*_FINAL` variants enable stricter/faster checks by marking leaves in the type hierarchy.

### 4.2 Type index allocation

At first access, registration helpers eventually call `TVMFFITypeGetOrAllocIndex` to:

1. allocate or reuse a type index;
2. record parent type relation and child-slot constraints;
3. install `TVMFFITypeInfo` metadata in `TypeTable`.

Built-in ABI types come from `TVMFFITypeIndex`; user types occupy dynamic ranges.

### 4.3 Reflection registration for classes

`tvm::ffi::reflection::ObjectDef<T>` is used in static init blocks:

- constructor: `.def(init<Args...>())`
- read-only field: `.def_ro("field", &T::field, ...)`
- read-write field: `.def_rw("field", &T::field, ...)`
- member/static methods: `.def(...)` / `.def_static(...)`

It emits metadata and accessors into runtime tables via reflection C APIs.

Related APIs:

- `FieldGetter`, `FieldSetter` for field-level access indirection;
- `ObjectCreator` for metadata-driven construction;
- `TypeAttrDef` / `TypeAttrColumn` for per-type attributes.

### 4.4 Global functions and overloads

Global function registration uses `reflection::GlobalDef(...)` and/or helper macros.

For overloaded callable surfaces, `OverloadObjectDef` builds dispatcher objects that choose
an implementation by argument signatures at runtime.

`src/ffi/function.cc` stores global entries in `GlobalFunctionTable`, and C APIs such as
`TVMFFIFunctionRegisterGlobal` / `TVMFFIFunctionGetGlobal` expose lookup and registration.

### 4.5 Static initialization strategy

`TVM_FFI_STATIC_INIT_BLOCK()` is the standard pattern for registration side effects.
This keeps declaration sites close to implementation code and ensures registration happens
during module load, before user calls into exported symbols.

`src/ffi/init_once.cc` provides one-time initialization utilities (`TVMFFIHandleInitOnce`)
for safe lazy init patterns.

## 5. Function object model and call pipeline

### 5.1 Dual call forms (`cpp_call` + `safe_call`)

`FunctionObj` stores callables in two compatible forms:

- `cpp_call`: C++-native callable path (throws C++ exceptions on failure).
- `safe_call`: C ABI path (`TVMFFISafeCallType`) returning integer status code.

This design allows:

- high-level C++ code to use exception-based ergonomics;
- foreign runtimes to use `int` + TLS error object style.

### 5.2 Packed argument convention

All cross-language invocations are packed through `TVMFFIAny` arrays:

- inputs: pointer + length to argument slice,
- output: one `TVMFFIAny` slot,
- errors: out-of-band via thread-local raised error object.

C++ wrappers (`AnyView`, `Any`) convert between native types and packed ABI values through
`TypeTraits`.

### 5.3 Typed wrappers

On top of packed functions, TVM-FFI provides typed wrappers (`TypedFunction`, macro helpers)
to give compile-time signatures in C++ while preserving packed ABI compatibility underneath.

## 6. Any/TypeTraits conversion model

### 6.1 `AnyView` vs `Any`

- `AnyView`: non-owning view over a `TVMFFIAny` payload.
- `Any`: owning value with RAII behavior for reference-managed payloads.

### 6.2 Conversion mechanism

`TypeTraits<T>` maps C++ types to ABI representations and back:

- arithmetic / POD-like values map to scalar union fields;
- object references map to `v_obj` + type index;
- container/object wrappers can define custom conversion behavior.

`ObjectRefWithFallbackTraitsBase` supports implicit conversion fallback patterns used by
high-level wrapper types.

### 6.3 Move-only transfer intent

`RValueRef<T>` exists to explicitly model move semantics through the packed call layer,
avoiding accidental copies and clarifying ownership handoff at API boundaries.

## 7. Module and dynamic library model

`ModuleObj` defines the abstract runtime module interface:

- get function by name
- query/import other modules
- optional binary serialization hooks

Concrete loading path for shared libraries:

1. `ffi.ModuleLoadFromFile` dispatches by file format / extension.
2. `.so`/dynamic-lib path goes through `library_module_dynamic_lib.cc`.
3. OS-specific loader (`dlopen`/`LoadLibraryW`) resolves exported symbols.
4. Runtime wraps the loaded library as `LibraryModuleObj`.

Rust/Python bindings call into this same module runtime via global functions.

## 8. Error model and exception bridge

C ABI calls return `0` on success and `-1` on failure (conventional TVM-FFI contract).

On failure:

1. C++ side catches exceptions/errors and materializes a `tvm::ffi::Error` object.
2. Error is stored in thread-local raised slot (`TVMFFIErrorSetRaised`).
3. Caller retrieves it (`TVMFFIErrorMoveFromRaised`) and maps it to language-native error.

This decouples ABI status signaling from rich diagnostic payloads.

## 9. End-to-end registration flow (class + method)

Typical flow for a user-defined C++ object:

1. Define `MyObj : Object` + declare type info macros.
2. Define `MyRef : ObjectRef` wrapper type.
3. In `TVM_FFI_STATIC_INIT_BLOCK()`, call `ObjectDef<MyObj>()...def(...).def_rw(...)`.
4. Library load triggers static registration.
5. Runtime type table and reflection tables receive metadata.
6. Other languages discover constructors/fields/methods through reflection APIs.
7. Calls eventually route into `FunctionObj` packed invocations with `Any` conversions.

## 10. Implementation constraints and tradeoffs

1. **Single inheritance only** in object hierarchy (runtime type table assumption).
2. **Intrusive refcount** lowers overhead but requires strict construction/layout discipline.
3. **Packed call ABI** is flexible but requires robust conversion/validation to keep typed
   wrappers safe.
4. **Static init registration** is ergonomic but requires care for initialization order and
   symbol visibility in shared libraries.
5. **Custom RTTI system** improves cross-language consistency but adds type-table complexity.

## 11. Practical extension checklist

When adding a new object family:

1. Add object class + type declaration macros.
2. Add object reference wrapper if user-facing.
3. Register reflection metadata in static init block.
4. Add tests for:
   - type checks/casting,
   - constructor/method/field reflection,
   - packed call conversions and error paths.
5. If exposed cross-language, verify:
   - Python/Rust wrapper discoverability,
   - module export and symbol loading behavior.

When adding a new function API:

1. Decide packed signature and type conversions.
2. Register as global and/or object method.
3. Add negative-path tests (conversion mismatch, thrown error, null handles).
4. Validate behavior through C ABI calls, not only C++ typed wrappers.
