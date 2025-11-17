# Thoughts on "out"-pointer, Nov 12 2025


_None of the following syntax proposals is binding so that a syntax proposal can be delayed as much as possible. This is a reconstruction of the conversation from the [Zulip chat](https://rust-lang.zulipchat.com/#narrow/channel/528918-t-lang.2Fin-place-init/topic/out-pointer.20type.20and.20MIR.20semantics.20consideration/with/541880132)._

_This document is supposed to cover some contents from [another dated Sep 18, 2025](https://hackmd.io/AmPxDXSFS9m3rAAaoGbz8w?edit). Contact wieDasDing@ if you find deficiency._

# Table of contents
[toc]

# `uninit`-pointer, formerly known as out-pointer

## Design constraints

To begin with, we would like to set out a list of constraints that a solution for `uninit`-pointer should meet.

- **First class type support**: one should be free to pass around `uninit`-pointers across function boundary through arguments.
- **Statically determinted initialised state**: the mechanism should allow the compiler to statically determine that at any given program point, whether sub-places behind an `uninit`-pointer are initialised.


## Sneak peek

Here we have a few worked examples.

### Kernel use-case

In this exercise, we demonstrate how `kernel::sync::lock::Mutex` will be constructed.

We start with `Opaque` in-place initialisation in the `kernel` crate.
```rust
// First, bindings from the C-side
mod bindings {
    pub type mutex;

    pub unsafe fn __mutex_init(ptr: *mut mutex);
}
#[repr(transparent)]
pub struct Opaque<T> {
    value: UnsafeCell<MaybeUninit<T>>,
    _pin: PhantomPinned, // should be replaced with `UnsafePinned`
}

impl<T> Opaque<T> {
    pub fn ffi_init(&uninit self, init_func: FnOnce(*mut T)) -> &own Self {
        init_func(self as *mut self);
        assume_init(self)
    }
    
    pub fn try_ffi_init<E>(
        &uninit self,
        init_func: FnOnce(*mut T) -> Result<(), E>
    ) -> Result<&own Self, E> {
        init_func(self as *mut self)?;
        Ok(assume_init(self))
    }
}
// For illustration purpose, we conjure `PinInit` as analogy to
// the `PinInit` trait from `pin_init` crate,
// except this is not part of our proposal yet.
// We also add support for UnsafeCell with this new trait.
pub trait PinInit {
    type Target;
    type Error;
    fn try_init(self, dest: &uninit Self::Target)
        -> Result<&own Self::Target, Self::Error>;
}
impl<T> UnsafeCell<T> {
    pub fn pin_init<PI: PinInit>(&uninit self, value: PI)
        -> Result<&own Self, PI::Error>
    {
        self.value <- value.try_init(&uninit self.value)?;
        Ok(self)
    }
}
```

```
Taylor's thoughts:
user calls method? on UnsafeCell<T> to transform:
  &uninit UnsafeCell<T> -> &uninit T
user transforms &uninit T to &own T
user calls method? on UnsafeCell<T> to transform:
  &own T -> &own UnsafeCell<T>
  
// <- allows `value` to be in scope on the rhs?
let value <- other.try_init(&uninit value)?;
//
let value;
value <- other.try_init(&uninit value)?;
// 
let point: Point;
point.x = 0;
point.y = 0;
// point is now initialized

// Taylor:
let point <- Point {
  some_ptr_field_pointing_to_self: &raw mut point,
};
```

So given that we define the `Mutex` as such.
```rust
#[pin_data]
pub struct Mutex<T> {
    #[pin]
    mutex: Opaque<bindings::mutex>,
    #[pin]
    value: UnsafeCell<T>,
}
```

We shall initialise any `Mutex<_>` as the following.
```rust
impl<T> Mutex<T> {
    pub fn new<PI: PinInit<Target = T>>(&uninit self, value: PI)
        -> Result<&own Self, PI::Error>
    where
        E: From<Infallible>,
    {
        self.mutex <- Opaque::ffi_init(
            &uninit self.mutex,
            |ptr| unsafe {
                bindings::__mutex_init(ptr);
            }
        );
        self.value <- value.try_init(&uninit self.value)?;
        Ok(self)
    }
}
```

### The monster struct

Here is an abridged version of [Asahi's monster struct](https://github.com/AsahiLinux/linux/blob/5ce014f7fa18021227bf46bd155b4ccf375a9dd3/drivers/gpu/drm/asahi/initdata.rs#L236-L478). Note that this is not the "scariest" of all. In practice bigger `struct`s in the wild are known to exist.
```rust
let output;
output <- raw::HwDataA::ver {
    unk_d20 <- f32!(65536.0), 
    unk_e10_0 <- {
        let filter_a = f32!(1.0) / pwr.se_filter_time_constant.into();
        raw::HwDataA130Extra {
            unk_58: 0x1,
            gpu_se_filter_a_neg: f32!(1.0) - filter_a,
            // ...
        }
    },
    // Taylor: it would be cool to be able to promise to LLVM
    // that particular memory has already been zeroed (e.g. via
    // a zero page mapped into virtual memory or via calloc).
    ..Zeroable::zeroable()
};
```

#### Notable features

We [notarise](#notarisation) a place behind an `&uninit T` into `&own T` with a `<-` statement. The type of the statement shall be `()`.
```rust
output <- init_output(&uninit output);
// or
let owned: &own T = init_output(&uninit output);
output <- owned;
```

In case the right hand side is `struct` literal, the right hand side is an "init" expression or an `&uninit _`, analogous to the notarisation statement.
```rust
output <- MyStruct {
    field <- $init_expr
};
```

We have packed initialisation, demanding the source expression to have a type `I: Init` or `TI: TryInit<T, E>`.
```rust
output <- MyStruct {
    ..Zeroable::zeroable()
};
```

### Interaction with place pattern and place expression

We propose to extend notarisation to [place pattern or place expression as well](https://doc.rust-lang.org/reference/expressions.html#r-expr.place-value), so that the following works, too.

```rust
fn init_two_of_them<'a, 'b>(a: &'a uninit A, b: &'b uninit B)
-> (&'a own A, &'b own B);

(a, b) <- init_two_of_them(&uninit a, &uninit b);
```

#### {Optional} Extension to `let`
The `let` statements at all possible places can be used for in-place initialisation, too.
```rust
let (a, b): $ascription <- init_two_of_them(&uninit a, &uninit b);
while let (a, b) <- init_two_of_them(&uninit a, &uninit b) {
    // a, b are in-place initialised
}

let MyStruct { a: a, b: b } <- init_two_of_them(&uninit a, &uninit b);
// .. or abridged as
let MyStruct { a, b } <- init_two_of_them(&uninit a, &uninit b);
```

## Summary of proposed syntax

Here is a worked example featuring all the proposed syntatx under this proposal.

```rust
impl Point {
    fn init(&uninit self) -> &own Self {
        self.x <- init_i32(&uninit self.x);
        self.y <- init_i32(&uninit self.y);
        self
    }
    
    fn init(&uninit self) -> &own Self {
        (self.x, self.y) <- init_i32_tuple(&uninit self.x, &uninit self.y);
        self
    }
    
    fn init(&uninit self) -> &own Self {
        self <- Self {
            x <- init_i32(&uninit x),
            y <- init_i32(&uninit y),
        };
        self
    }
    
    fn init(&uninit self) -> &own Self {
        // you could also do
        self.x <- init_i32(&uninit self.x);
        self.y <- init_i32(&uninit self.y);
        self
    }
}

fn main() {
    let x <- Struct {
        inner: Default::default(),
        ..Default::default(),
    };
}

impl Point {
    fn init(&uninit self) -> &own Self {
        (self.x, self.y) <- weird(&uninit self.x, &uninit self.x, &uninit self.y);
        self
    }
}
```

<details>
    <summary>Additional proposal</summary>
    Ding: This idea was proposed, but I haven't figured out the typing.
    
    
    fn init(&uninit self) -> &own Self {
        // or
        Self {
            x <- init_i32(&uninit x),
            x <- init_i32(x),
            y <- init_i32(&uninit y),
        }
        
        // compare with pin-init today:
        /*
        pin_init!(Self {
            x <- init_i32(),
            y <- init_i32(),
        })
        */
    }
</details>

## Type consideration

### Notations

#### Subtyping on regions
The following notation is adopted to describe region relations.

- When we want to say that live range of `'a` contains that of `'b`, we write as a predicate `'a: 'b` as per Rust language `where` clause syntax, or `'a :> 'b`;
- Conversely, when live range of `'b` contains that of `'a`, we write `'b: 'a` or `'a <: 'b`;
- When both of the forementioned condition hold, it is written as `'a <:> 'b` for short;
- When neither of the forementioned condition holds, it is written as `'a </> 'b`, or **"incomparable" to each other**.

#### Subtyping on types
The notation is extended to types as well, in the way that the types must match modulo region, and the regions in the type signatures are compared, respecting the ambient variance. As an example, `T<'a> <: U<'b>` implies `'a :> 'b` given that `'?0` in `T<'?0>` is in a contravariant position.

### Type `&'a uninit T`

An `uninit`-pointer is a reference qualified with `uninit`.

```rust
uninit_pointer: &'a uninit Type
```

The type of referent of an out-pointer must be `Sized` and one of ADT or tuples.

```rust
struct Type {
    field: u8,
}
let uninit_place: Type;
let uninit_pointer: &uninit Type = &uninit uninit_place;
// Then `Type: Sized` and is either an ADT or a tuple
// and additionally `Type` is local in the current crate
// and its fields are visible to the current body
```

##### Variance environment of `&'a uninit T`

The `'a` in `&'a uninit T` is in an invariant position. `&'a uninit T <: &'b uninit U` necessarily implies `'a <:> 'b`.

### Initialisation certificate, `&own T`
When we lend out a field out of a `&uninit` reference to a sub-procedure like functions, closures and coroutines, we need a mechanism for the sub-procedure to certify that the entirety of the lent out place behind an `&uninit` is fully initialised. This will enable the caller of the sub-procedure to **discharge** the uninitialisation state of the lent out place.

In favour of expressiveness, we propose that the "certificate" has a type representation in Rust type system. The reason being is that the ability to write function pointer signature like `fn() -> ?` and trait predicates like `FnMut() -> ?` where `?` is the certificate type is very important. This makes out-pointer a more composable abstraction and, thus, a more useful one.

In order to establish a relationship between an initialisation certificate and a corresponding specific `&'a uninit T` instance, the type has to encode information so that the caller can re-constitute this connection. Our proposal here is to include the lifetime `'a` of the the out-pointer. Let us denote the type as `&'a own T`. When a `&'a uninit T1` is discharged with its corresponding certificate `&'b own T2`, `'a <:> 'b` and `T1 <:> T2` must hold.

This certifying type must be able encode its unique relationship with exactly one of the `&uninit` inputs into the function.

Here are the basic facts about this type `&'a own T`:
- it is inhabited;
- `T: 'a`;
- it is transparently a raw pointer pointing at the place behind a corresponding `&'a uninit T` pointer;
- it implements `auto` traits through the constituent type `T` but never `Clone` or `Copy`, so that it is non-fungible;
- the place behind `&own T` can be mutably borrowed, so that `&mut *&own value: &mut T` for `value: T`;
- it has a destructor that delegates the destructor of `T`.

##### Variance environment of `&'a own T`

The lifetime `'a` at its position in the signature is invariant. Specifically, from `&'a own T <: &'b own T` we can deduce that `'a <:> 'b`

## MIR, region and borrow checking considerations

### Region rules involved in introduction of `&uninit T`

During borrow-checking, the numbered lifetime `'?a` assigned to a newly introduced `&'?a uninit` should have the constraint `'?a <: T`, `'?a lives-at $current_program_point`, like a regular MIR `Rvalue::Ref` would do to its lifetime.

This also applies to `uninit`-reborrowing, bzw `_a = &uninit (*_b)`, with or without projections, with the subregion relations populated in the same way as the plain-old references.

#### Initialisation state of the place behind the `uninit`-borrow

The initialisation state of the place behind the `uninit`-borrow should be statically determined to be **uninitialised** as known to the borrow checking.

Depending on the state of the `uninit`-borrow, we have these scenarios.
- If it is known to the borrow checker as always being initialised, the in-place drop should be invoked on thie place behind before the `uninit`-borrow is generated. In particular, when the place behind is getting re-assigned or re-in-place-initialised, the drop should be invoked.
```mir
// case 1
_uninit = &uninit _place;
_own = initialise() -> [return: bb1, ..];

bb1: {
_place <- _own;
// `_place` is initialised ..
_uninit = &uninit _place; // so a drop on `_place` should be called before
}
```

- If it is known to the borrow checker that at the current statement `_another_borrow = &uninit _place`
    * a place `_place` is already under some borrow, be it `uninit`- or immutable- or mutable-borrow;
    * the borrow in question is live, like in particular a `&own` with matching region,

  then an access conflict error should be emitted.
  ```mir
  _uninit = &uninit _place;
  _uninit2 = &uninit _place; // access conflict, `_uninit` is still `uninit`-borrowed
  // the only allowed `uninit`-borrow involving `_place` is reborrowing `_place`,
  // like either ..
  _uninit_field = &uninit (*_uninit).field;
  // .. or "reborrow" of whole
  _uninit2 = &uninit (*_uninit);
  ```

  :construction: _Call for experiment: the conflict error message needs design through experimentation._

- If it is known to the borrow checker that at the current statement `_place <- _own` where `_own: &'a own T` holds necessarily
    * a place `_place` is already under some borrow, be it `uninit`- or immutable- or mutable-borrow;
    * the borrow in question is live, like in particular a `&own`, but the live range of the borrow does not match with the region value `'a`,
  
  then an access conflict error should also be emitted.

  :construction: _Call for experiment: the conflict error message needs design through experimentation._

The MIR builder should introduce drop flags on those borrows when the initialisation state of the place in interest cannot be statically determined.

### <a id="notarisation"/> Region rules involved in notarisation of a place

When we notarise a place with a region `'a` with `&'b own T`, we demand `'a <:> 'b`. By notarisation, we certify to the borrow checker that the place and its descendent places as fully-initialised.

We need to prevent confusing the notarisation one `&uninit T` by smuggling another `&own T`. Otherwise, we would notarise the wrong place leading to potential unsoundness in the following example.

```rust
fn bad<'a, T>(_: &'a uninit T, _: &'a uninit T) -> &'a own T {
    // flip a coin, make a wish
}

let ua = &uninit a; // this has a new lifetime 'x
let ub = &uninit b; // this has a new lifetime 'y
ua <- bad(ua, ub);  // 'x and 'y shall not unify


// 1) we should allow reborrowing &'_ uninit T
let a: A;
let a0 = &'? 0 uninit a;
// if we allow ...
{ let b = &uninit a0.field; }
// ... we should allow reborrowing the whole
let a1 = &'? 1 uninit *a0;
// ... and if we allow it, what is the valuation of the anonymous region attached to `a1`?
// choice A: '?1 so that `'?1 <: '?0` ? Mkay.
// choice B: '?0, but then a1 dominates a0 but live range of a1 is contained in that of a0, so borrowck will not be happy.
// 2) discharging &'_ uninit T without initialisation should be allowed
fn try_init<'a>(a: &'a uninit A) -> Result<&'a own A, ()> {
    if flip_a_coin() {
        a.field = 1;
        Ok(a)
    } else {
        // here we leave `a` especially `*a` uninitialised
        // but it is fine
        Err(())
    }
}

struct Evil<'a>(&'a uninit A, &'a uninit A);
fn merge<'a, 'b: 'a>(a: &'a uninit A, b: &'b uninit A) -> Evil<'a>
{
    // so we can reborrow here right?
    Evil(&uninit *a, &uninit *b)
}
let a: A;
let b: A;
// this should not work because ...
let Evil(a, b) = merge(&uninit a, &uninit b);
// the two `&uninit`s are live for two different sets of program points
// because `&uninit a` and `&uninit b` are necessarily instantiated
// in two different MIR statements
a <- bad(a, b); // therefore, the call to `bad` does not type-check
// because `'a` and `'b` cannot be unified as one late-bound lifetime parameter

// this also should not work:
let x = &uninit x;
let ua = &uninit x.a;
let ub = &uninit x.b;
ua <- bad(ua, ub);
```

### Coercion of `&uninit T` to `&own T`

A `&'a uninit T` should be transparently coerced into `&'b own T` where `'a <:> 'b`, when the borrow checker can check and assert that the borrowed place tagged with the region `'a` has be notarised or statically known to be initialised.

```rust
fn init_me(data: &uninit Data) -> &own Data {
    if CONDITION {
        return data; //~ ERROR: `*data` may not be fully initialised
        // therefore, the coercion is rejected
    }
    data.inner = new_inner();
    data // coercion happens because `*data` is fully initialised
    // Expectation: the return value of type `&own Data` certifies
    // the initialisation already,
    // so no drop is invoked on `*data`.
}
```

At coercion site, the place behind an `&uninit T`, such as `*data` in the example, will be temporarily switched back to the uninitialised state from the borrow checker's perspective: the resultant `&own T` witnesses the initialisation.

### Asserted notarisation

An `unsafe` intrinsic, as an MIR `Rvalue`, should be available for asserting the notarisation of a `&'a uninit T`, returning `&'b own T`, with `'a <:> 'b`. This is useful for interoperability when the in-place initialisation is typically handed-off to a foreign function or process.

The proposed intrinsic `assume_init` is the following.

```rust
let x: T;
x <- unsafe {
    call_ffi(&uninit x as *mut T);
    // Safety: `x` is not statically known to be initialised
    // after the `call_ffi` returns,
    // so it is safe to `uninit`-borrow `x` again
    // without invoking the `T` destructor.
    assume_init(&uninit x)
};
```

### :construction: Extension: Pattern-matching

We also intend to support pattern-matching on `&uninit T`s. From the surface syntax, we would like to support the following pattern-matching semantics.

#### `let` statement

`let` statement shall possibly de-initialise the place under the `uninit`-borrow mode and, transition the place into partially moved state and set discriminant on that place.

```rust
let Struct::Variant { ref uninit a, b } = &uninit place;
// .. de-initialises `place` and set the discriminant to that of `Variant`
// while projecting out `a: &uninit A` and `b: &uninit B`
// **from two MIR statements**.
```

## Library consideration

### Supporting trait `Init<T>`
We would like to introduce supporting type with syntax support.

```rust
#[lang_item = "init_trait"]
pub trait Init<T> {
    fn init(self, dest: &uninit T) -> &own T
}
```

### Auxiliary trait `TryInit<T>`
In addition to `Init`, we propose to further introduce an auxiliary traits `TryInit<T>`.

```rust
#![feature(try_trait_v2)]
#[lang_item = "try_init_trait"]
pub trait TryInit<T> {
    type Output: Try<Output = &'a own T>
        where Self: 'a, T: 'a;
    
    fn try_init<'a>(self, dest: &'a uninit T)
        -> Self::Output<'a>;
}

pub trait TryInitToResult<T>: TryInit<T>
where
    for<'a> Result<&'a own T, Self::Error>: FromResidual<Self::Output<'a>>,
{
    type Error;
}
```

## Syntax consideration

### Basics of emplacement statement `<-` and the try-variant `<-?`

There are two modus operandi of `<-`. First is the notarisaion of a place expression or place pattern by emplacing an ADT literal, tuple, repeat or array expression.

#### Emplacement context

The right-hand side **source** `expr` of an emplacement statement `place <- expr;` is considered inside an emplacement context in a **target** place `place`. The HIR type checking of the statement would perform a sequence of trials to determine the emplacement action.

1. When the **source** expression has a type known to implement `Init<T>`, the emplacement action would be that the **target** place would be coerced into `T` with no adjustments permitted, followed by an `uninit`-borrow as the argument to the invocation of `Init<T>::init` call, and finally notarisation of the **target** place with the returned value.
```rust
fn ctor() -> impl Init<Data>;
let data: Data;
// because the target is a `Data` and the source produces an `impl Init<Data>` ...
data <- ctor();
// the emplacement action is invocation of `Init::init`
```
2. When the **source** expression in the emplacement statement has a concrete type matching the concrete type of the **target** place, the emplacement action is direct assignment of the evaluation result into the target place.
```rust
fn make_data() -> Data;
let data: Data;
// because the target is a `Data` and the source produces a `Data` ...
data <- make_data();
// the emplacement action is a direct assignment
```

#### Convenient fallible emplacement through `<-?` and `TryInit`

Emplacement can fail during the initialisation process. Therefore, earlier we proposed `TryInit<T>` and we would like to propose the counter-part of `<-` for the fallible case.

```rust
struct Field {}
struct Root {
    data: Field,
}
fn ctor() -> impl TryInitIntoResult<Field, Output = Error>;

fn maybe_init(a: &uninit Root) -> Result<&own Root, Error> {
    a.data <-? ctor();
    Ok(a)
}
```

##### Extension: convenient adaptor for use of `&uninit` constructor calls as `Init`

It is observed that use of `&uninit T` and `&own T` often involves immediate notarisation upon completion of an emplacing constructor call, with the drawback of naming the one and only initialising place twice, like the following example.

```rust
fn ctor(_: &uninit A, b: B) -> &own A;

let a: A;
a <- ctor(&uninit a, b); // `a` is named twice
```

To reduce boilerplate and improve readability for single `&uninit`/`&own` pair case, we would like to propose a builtin macro `emplace!` that generates a `Init<T>` object along with necessary captures for emplacement action.

Conceptually the macro would be prototyped as follows. The macro `emplace` will replace a syntatical sigil in the input function call argument, the `@` symbol as we propose, with the `&uninit` slot. So that the example can be translated as follows.
```rust
a <- emplace! { ctor(@, b) };
// or
let ctor = emplace! { ctor(@, b) };
a <- ctor;

// which is equivalent to ..
struct Ctor(B);
impl Init<A> for Ctor {
    fn init(self, out: &uninit A) -> &own A
    {
        ctor(out, self.0)
    }
}
a <- Ctor(b);
```

##### :construction: Future consideration: emplacing closure

There has been interest in writing objects implementing `TryInit`, `TryInitIntoResult` conveniently with a closure syntax. We would like to delay the syntax consideration to a future point of time. However, the `try_init!` macro from `pin_init` crate provides sufficient inspiration on the syntax construction.

#### Tuples, repeats, arrays and primitives literals

Tuples are conceptual anonymous structs while arrays and repeats . Therefore, these notation on the right-hand side of a `<-` statement receive special treatment, so that the elements are initialised in the emplacement context.
```rust
// `data1, `data2` and `data3` are evaluated in the emplacement context
place <- (data1, data2, data3);
place <- [data1, data2, data3];
place <- [data1; N];
```

Primitives literals subject to const promotion like `0u8`, `"str"` and constant expression such as `const { .. }` can also be in-place initialised, when they do not fall under the tuples, repeats or arrays case.
```rust
a <- "hello world";
b <- T::AssociatedConst;
c <- 1 + 1;
```

Due to code-generation consideration, we have found this kind of use unadvisable. Our recommendation is elaborated in the code-generation section.

#### ADT literal

We propose the following syntax for ADT literals.
```rust
let data: Data = ..;
let init_data: impl Init<Data> = .. ;
// ADT literal
place <- Struct::Variant {
    data, // `data` is moved into the destination
    data_field <- init_data, // `init_data.init()` in-place construction is invoked 
    ..Default::default_init()
};
```

Here are the remarkable features of handling ADT literals.
##### Direct assignment
Field specification `a: b` works like direct assignment.
```rust
place <- Struct {
    a: b,
};
// is equivalent to
place.a = b;
```
##### Short-hand
Like plain-old ADT literals, we would support single occurrence of identifier `a` when the field name `a` and the emplacement context expression as a local `a` coincides. It is equivalent to `a <- a`.
```rust
place <- Struct::Variant {
    a,
};
// is equivalent to
place <- Struct::Variant {
    a <- a
};
```
##### Aggregated emplacement
We would like to support a pack of fields in the target ADT data, without the mandatory exhaustiveness in ADT fields and with an optional default expression after the `..` sigil.
```rust
place <- Struct::Variant {
    field1,
    ..Default::default_init()
}
// This `field2` can be initialised
// outside the aggregated emplacement:
place.field2 <- field2();
```
Here `Default::default_init()` shall implement `Init<Struct>`. Its evaluation comes at the first before all the other fields in the aggregate.

##### Lexical scoping
In an aggregate emplacement, it is possible that fields may use previously initialised fields. We would allow this by exposing the initialisation state of previous fields to the next fields.
```rust
place <- Struct::Variant {
    field1,
    // `place.field1` is considered initialised
    field2 <- make_ref(&place.field1),
};
```

#### Others
We still support other kinds of expression on the right-hand side, when it fails to fit in the previous categories due to possibly unfulfilled trait relationship, and interpret the statement as direct assignment.

## Code generation consideration

It is observed that in some cases, specifically when unwinding on panics is concerned, a `&own T` pointer may be superfluous because its only purpose is to notarise initialisation, while its content as an address is left unread. One may note that this is a typical candidate of dead value elimination. We would advise the implementation to enable such elimination as much as possible, probably by favouring inlining of functions returning `&own T` as data.

### :construction: {Optional} Lint against superfluous in-place initialisation

We advise against in-place initialisation on primitive types other than references and slices; or tuples or ADTs containing only those primitive types so that the size of the type is below a target-dependent size. Direct assignments in this case are almost always better than in-place initialisation thanks to good code generation and target architecture capability while in-place initialisation has to be necessarily materialised into memory writes and can be inferior in performance and hurt readability. We propose a lint to recognise the notarisation statements `<-` from user program, not including macro expansions, where the type of the target place falls in this category.

## FAQs

### Why `&own T`?

This is the first time that Rust would need to "communicate" initialisation state across subroutine boundary. There is no type-level signal that today's Rust can use to manipulate the initialisation state of places in the caller context. In various designs, the need to manipulate the init state will eventually manifest itself in one form or another. However, we choose `&own T` because of these two reasons.
- It is guardrail for ensuring proper destruction, in case it is left unused for notarising the source init place.
- It is basically a reference to the initialised place, and we have intention and future extension to utilise the fact that it is a reference.

### Can we get "smarter" about notarising the initialisation state, from the return values containing `&own T`?


```rust
fn method() -> T {
    let a: T;
    match get_something_init(&out a) {
        Ok(cert) => {
            return a
        }
    }
}
```

```rust
fn method() -> T {
    let a: T;
    match get_something_init(&out a) {
        Ok(cert) => {
            return a
        }
    }
}
```


# Important consideration - the MIR place evolution

_[Zulip chat](https://rust-lang.zulipchat.com/#narrow/channel/213817-t-lang/topic/Field.20projections.20and.20places/with/544815635)_

# Initializing through types

## Types that contain values
```rust
// Allows coercing a `&own Uninit<Me<T>>` to `&own Me<Uninit<T>>` when only fields
// mentioning `T` are uninitialized.
unsafe trait CoerceUninit {}
```


# Action Items

-  ~~`LHS <- RHS` mentions place twice, can we do better?~~

- Should we go for `<-` in struct initaliser or coercion when `:` is used?

- Fn sig of constructor of a struct that delegates to other in-place initialisers? How to make transparent data like `UnsafeCell` play nice with its counterpart `&uninit UnsafeCell`?