# Thoughts on In-place init, Sep 18 2025

# out-pointer

_None of the following syntax proposals is binding so that a syntax proposal can be delayed as much as possible. This is a reconstruction of the conversation during Kangrejos 2025._

So we have confirmed that dataflow analysis can be adapted to partially support the [partial initialisation in borrow checker](https://github.com/rust-lang/rust/issues/54987).

We would like to extend the idea to out-places and out-pointers.

## Notation

An out-place is a user local qualified with `out`.

```rust
let out uninit_place: Type;
```

An out-pointer is a reference qualified with `out`.

```rust
uninit_pointer: &'a out Type
```

### Basic types

The type of an out-place and referent of an out-pointer must be `Sized` and one of ADT or tuples, *without `#![non_exhaustive]` unless the type is local in the current crate and visible to the current body*.

```rust
#![non_exhaustive]
struct Type {
    field: u8,
}
let out uninit_place: Type;
let uninit_pointer: &out Type = &out uninit_place;
// Then `Type: Sized` and is either an ADT or a tuple
// and additionally `Type` is local in the current crate
// and its fields are visible to the current body
```

#### Well-formedness and auto traits
For `&'a out T` to be well-formed, it is necessary that `T: 'a + Sized`.

`&out T` is never `Sync`.

`&out T` is `Send` if `T` is `Sync`.

`&out T` is `Unpin`.

#### Initialisation certificate
When we lend out a field out of an `out` variable or `&out` reference to a sub-procedure like functions, closures and coroutines, we need a mechanism for the sub-procedure to certify that the entirety of the lent out place behind an `&out` is fully initialised. This will enable the caller of the sub-procedure to **discharge** the uninitialisation state of the lent out place.

In favour of expressiveness, we propose that the "certificate" has a type representation in Rust type system. The reason being is that the ability to write function pointer signature like `fn() -> ?` and trait predicates like `FnMut() -> ?` where `?` is the certificate type is very important. This makes out-pointer a more composable abstraction and, thus, a more useful one.

In order to establish a relationship between an initialisation certificate and a corresponding specific `&'a out T` instance, the type has to encode information so that the caller can re-constitute this tie. Our proposal here is to include the lifetime `'a` of the the out-pointer. Let us denote the type as `init<'a>`. When a `&'a out T` is discharged with its corresponding certificate `init<'b>`, `'a` and `'b` must be invariant.

This certifying type must be able encode its unique relationship with exactly one of the `&out` inputs into the function.

Here are the basic facts about this type:
- it is inhabited;
- it is 1-ZST;
- it implements `Send + Sync` but never `Clone` or `Copy`, so that it is non-fungible;
- a user inherent or trait implementation is never allowed;
- it has a trivial destructor.


#### Region rules involved in 

When the arguments of a function signature contains two out-pointers, `&'a out X` and `&'b out Y`, the region `'a` and `'b` must be bivariant to each other in order for the function signature to be considered well-formed. To check this, the relations `'a :> 'b` and `'b :> 'a` are tested and the well-formedness is checked if both relations yield no solution. This rule is required so that we can safely establish a unique map between a given `&'a out T` passed in as an input and an `init<'a>` value with no ambiguity.

#### Further consideration

It is possible that `init<'a>` might be used as data in other types.

```rust
type XandY<'a, 'b> = (init<'a>, init<'b>);
// XandY<'a, 'b> is well-formed only if the generics 'a and 'b are bivariant

fn initialize<'a, 'b>(x: &'a out X, y: &'b out Y)
    -> Result<XandY<'a, 'b>, Error>
// XandY<'a, 'b> is well-formed only if 'a and 'b in the typing environment are bivariant
{
    ..
}
```

This might imply that we demand a new sets of requirements for types and ADTs to be well-formed. If a data type contains a field with `init<'a>`, where `'a` is a generic lifetime, it is a **rigid** lifetime generic parameter. The property of a rigid lifetime generic parameter is that within a same data type, when two rigid lifetime generics are instantiated with two region values, these two region values must be bivariant to each other, or in other words with no containment relationship in one way or another.

```rust
struct SomeType<'a>(init<'a>);
//              ^~ this generic parameter is rigid
struct Invalid<'a, 'b: 'a>(init<'a>, init<'b>);
//                 ^~~~~~            ^~~~~~~~ this is invalid because ...
//                 |_ 'b :> 'a is provable, but 'a and 'b are rigid
```

The same rule applies to function types, `fn(&'a out T) -> init<'a>` and `Fn*(&'a out T) -> init<'a>`. As an example, in checking `for<'a, 'b> fn(&'a out T, &'b out T, &'c out T) -> (init<'a>, init<'b>, init<'c>)`, since `'a` and `'b` will be assigned a universal placeholder value and, therefore, automatically bivariant to each other; `'c` lives in an outer universe and cannot name neither `'a` nor `'b` while neither `'a: 'c` nor `'b: 'c` holds. Therefore, this function pointer type is well-formed.


##### Unresolved question
What would this do to the other types? How would it work if `init<'a>` is captured by a closure, a coroutine or a coroutine closure which generates a coroutine of lifetime `'env`?

# Proposed MIR semantics

## Init states
The introduction of out-place would entail a relaxation of init states. Current MIR demands that projections out of a place are dominated by initialisation of the place as a whole. Our proposal requires this rule to be relaxed, so that assignments into possibly nested fields are allowed.

```rust
struct Type {
    a: u32,
    b: u32,
}
let out foo: Type;
foo.a = 0;
// use_me(&foo.b): ERROR, foo.b is uninit
// use_me(&foo): ERROR: foo.b is uninit
foo.b = 1;
// both a and b are now path-initialised
use_me(&foo.b); // OK
use_me(&foo); // OK
```

This code is still lowered as if `out foo` is "valid" for per-field assignments.

```mir
StorageLive(_foo)
_foo = const Type { 0: const 1u32, 1: const 2u32 };
_foo.0 = const 1u32;
_foo.1 = const 2u32;
_foo_b = &foo.b;
call use_me(move _foo_b) [return: .., unwind: ..]
```

## Function returning

At function returns, all `&out` and `out` variables must be discharged. It can be discharged either implicitly by inspecting initialisation state of the descendants of the `out` places and paths rooted in `&out` variables.

## MIR statements

We propose the following changes to MIR statements and their operational semantics.

### Statements

#### `Assign`

We assume the following local declarations with renumbered regions.
- `let _x: &'a out X;`
- `let _y: &'b out Y;`

```mir
_y = &out (*_x).y;
```
The pre-condition of constructing the `&out (*_x).y` is that the place `(*_x).y` must be uninitialised. This emits region relation `'b :> 'a`.

```mir
*y = Y { .. };
```
The primary effect here is the place `*y` is set to be initialised. However, `(*_x).y` is considered to be still uninitialised. To discharge it, we need to use `_init = AssertInit(*y)` and `SetInit((*_x).y, move _init)`

:warning: **Unresolved question**: What is the precondition of the assignment here? I suppose we need to assert uninitialised state.

#### `drop` terminator

`drop($place)` has a pre-condition now: `$place` must be fully initialised.
Its primary effect is that `$place` is uninit again, so that the place can be lend out with `&out $place` once more.

#### :sparkles: AssertInit(`<mir place>`) intrinsic

```mir
let _init: &'c out X;
_init = AssertInit(*_x);
```
This emits region relation `'a <:> 'c` so that they are invariant. The borrow checker must assert that `*_x` is fully initialised as a pre-condition.

#### :sparkles: AssumeInit(`<mir place>`) intrinsic

This is an unsafe intrinsic that is only emitted from an unsafe function call.

```mir
_init = AssumeInit(*_x);
```
The primary effect is that the place `*_x` is considered initialised.

#### :sparkles: SetInit(`<mir place>`, `<init>`)

```mir
SetInit(*_x, move _init);
```
The borrow checker must assert that `'a <:> 'c` as a pre-condition. The primary effect is that the place `*_x` is considered initialised and `_init` uninitialised.
