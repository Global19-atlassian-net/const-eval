- Feature Name: const_generic_const_fn_bounds
- Start Date: 2018-10-05
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow `impl const Trait` for trait impls where all method impls are checked as const fn.

Make it legal to declare trait bounds on generic parameters of const functions and allow
the body of the const fn to call methods on the generic parameters that have a `const` modifier
on their bound.

# Motivation
[motivation]: #motivation

Currently one can declare const fns with generic parameters, but one cannot add trait bounds to these
generic parameters. Thus one is not able to call methods on the generic parameters (or on objects of the
generic parameter type), because they are fully unconstrained.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

You can call methods of generic parameters of a const function, because they are implicitly assumed to be
`const fn`. For example, the `Add` trait bound can be used to call `Add::add` or `+` on the arguments
with that bound.

```rust
const fn triple_add<T: Add>(a: T, b: T, c: T) -> T {
    a + b + c
}
```

The obligation is passed to the caller of your `triple_add` function to supply a type whose `Add` impl is fully
`const`. Since `Add` only has `add` as a method, in this case one only needs to ensure that the `add` method is
`const`. Instead of adding a `const` modifier to all methods of a trait impl, the modifier is added to the entire
`impl` block:

```rust
struct MyInt(i8);
impl const Add for MyInt {
    fn add(self, other: Self) -> Self {
        MyInt(self.0 + other.0)
    }
}
```

The const requirement is inferred on all bounds of the impl and its methods,
so in the following `H` is required to have a const impl of `Hasher`, so that
methods on `state` are callable.

```rust
impl const Hash for MyInt {
    fn hash<H>(
        &self,
        state: &mut H,
    )
        where H: Hasher
    {
        state.write(&[self.0 as u8]);
    }
}
```

## Drop

A notable use case of `impl const` is defining `Drop` impls. If you write

```rust
struct SomeDropType<'a>(&'a Cell<u32>);
impl const Drop for SomeDropType {
    fn drop(&mut self) {
        self.0.set(self.0.get() - 1);
    }
}
```

Then you are allowed to actually let a value of `SomeDropType` get dropped within a constant
evaluation. This means `(SomeDropType(&Cell::new(42)), 42).1` is now allowed, because we can prove
that everything from the creation of the value to the destruction is const evaluable.

Note that all fields of types with a `const Drop` impl must have `const Drop` impls, too, as the
compiler will automatically generate `Drop::drop` calls to the fields:

```rust
struct Foo;
impl Drop for Foo { fn drop(&mut self) {} }
struct Bar(Foo);
impl const Drop for Foo { fn drop(&mut self) {} } // not allowed
```

## Runtime uses don't have `const` restrictions?

`impl const` blocks additionally generate impls that are not const if any generic
parameters are not const.

E.g.

```rust
impl<T: Add> const Add for Foo<T> {
    fn add(self, other: Self) -> Self {
        Foo(self.0 + other.0)
    }
}
```

allows calling `Foo(String::from("foo")) + Foo(String::from("bar"))` even though that is (at the time
of writing this RFC) most definitely not const, because `String` only has an `impl Add for String`
and not an `impl const Add for String`.

This goes in hand with the current scheme for const functions, which may also be called
at runtime with runtime arguments, but are checked for soundness as if they were called in
a const context. E.g. the following function may be called as
`add(String::from("foo"), String::from("bar"))` at runtime.

```rust
const fn add<T: Add>(a: T, b: T) -> T {
    a + b
}
```

This feature could have been added in the future in a backwards compatible manner, but without it
the use of `const` impls is very restricted for the generic types of the standard library due to
backwards compatibility.
Changing an impl to only allow generic types which have a `const` impl for their bounds would break
situations like the one described above.

## `?const` opt out

There is often desire to add bounds to a `const` function's generic arguments, without wanting to
call any of the methods on those generic bounds. Prominent examples are `new` functions:

```rust
struct Foo<T: Trait>(T);
const fn new<T: Trait>(t: T) -> Foo<T> {
    Foo(t)
}
```

Unfortunately, with the given syntax in this RFC, one can now only call the `new` function in a const
context if `T` has
an `impl const Trait for T { ... }`. Thus an opt-out similar to `?Sized` can be used:

```rust
struct Foo<T: Trait>(T);
const fn new<T: ?const Trait>(t: T) -> Foo<T> {
    Foo(t)
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The implementation of this RFC is (in contrast to some of its alternatives) mostly
changes around the syntax of the language (allowing `const` modifiers in a few places)
and ensuring that lowering to HIR and MIR keeps track of that.
The miri engine already fully supports calling methods on generic
bounds, there's just no way of declaring them. Checking methods for constness is already implemented
for inherent methods. The implementation will have to extend those checks to also run on methods
of `impl const` items.

## Implementation instructions

1. Add an `maybe_const` field to the AST's `TraitRef`
2. Adjust the Parser to support `?const` modifiers before trait bounds
3. Add an `maybe_const` field to the HIR's `TraitRef`
4. Adjust lowering to pass through the `maybe_const` field from AST to HIR
5. Add a a check to `librustc_typeck/check/wfcheck.rs` ensuring that no generic bounds
    in an `impl const` block have the `maybe_const` flag set
6. Feature gate instead of ban `Predicate::Trait` other than `Sized` in
    `librustc_mir/transform/qualify_min_const_fn.rs`
7. Remove the call in https://github.com/rust-lang/rust/blob/f8caa321c7c7214a6c5415e4b3694e65b4ff73a7/src/librustc_passes/ast_validation.rs#L306
8. Adjust the reference and the book to reflect these changes.

## Const type theory

This RFC was written after weighing practical issues against each other and finding the sweet spot
that supports most use cases, is sound and fairly intuitive to use. A different approach from a
type theoretical perspective started out with a much purer scheme, but, when exposed to the
constraints required, evolved to essentially the same scheme as this RFC. We thus feel confident
that this RFC is the minimal viable scheme for having bounds on generic parameters of const
functions. The discussion and evolution of the type theoretical scheme can be found
[here](https://github.com/rust-rfcs/const-eval/pull/8#issuecomment-452396020) and is only 12 posts
and a linked three page document long. It is left as an exercise to the reader to read the
discussion themselves.

# Drawbacks
[drawbacks]: #drawbacks

It is not a fully general design that supports every possible use case,
but it covers the most common cases. See also the alternatives.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Effect system

A fully powered effect system can allow us to do fine grained constness propagation
(or no propagation where undesirable). This is way out of scope in the near future
and this RFC is forward compatible to have its background impl be an effect system.

## Fine grained `const` annotations

One could annotate methods instead of impls, allowing just marking some method impls
as const fn. This would require some sort of "const bounds" in generic functions that
can be applied to specific methods. E.g. `where <T as Add>::add: const` or something of
the sort. This design is more complex than the current one and we'd probably want the
current one as sugar anyway

## Require `const` bounds everywhere

One could require `const` on the bounds (e.g. `T: const Trait`) instead of assuming constness for all
bounds. That design would not be forward compatible to allowing `const` trait bounds
on non-const functions, e.g. in

```rust
fn foo<T: const Bar>() -> i32 {
    const FOO: i32 = T::bar();
    FOO
}
```

## Infer all the things

We can just throw all this complexity out the door and allow calling any method on generic
parameters without an extra annotation `iff` that method satisfies `const fn`. So we'd still
annotate methods in trait impls, but we would not block calling a function on whether the
generic parameters fulfill some sort of constness rules. Instead we'd catch this during
const evaluation.

This is strictly the most powerful and generic variant, but is an enormous backwards compatibility
hazard as changing a const fn's body to suddenly call a method that it did not before can break
users of the function.

# Future work

This design is explicitly forward compatible to all future extensions the author could think
about. Notable mentions (see also the alternatives section):

* an effect system with a "notconst" effect
* const trait bounds on non-const functions allowing the use of the generic parameter in
  constant expressions in the body of the function or maybe even for array lenghts in the
  signature of the function
* fine grained bounds for single methods and their bounds

It might also be desirable to make the automatic `Fn*` impls on function types and pointers `const`.
This change should probably go in hand with allowing `const fn` pointers on const functions
that support being called (in contrast to regular function pointers).

## Deriving `impl const`

```rust
#[derive(Clone)]
pub struct Foo(Bar);

struct Bar;

impl const Clone for Bar {
    fn clone(&self) -> Self { Bar }
}
```

could theoretically have a scheme inferring `Foo`'s `Clone` impl to be `const`. If some time
later the `impl const Clone for Bar` (a private type) is changed to just `impl`, `Foo`'s `Clone`
impl would suddenly stop being `const`, without any visible change to the API. This should not
be allowed for the same reason as why we're not inferring `const` on functions: changes to private
things should not affect the constness of public things, because that is not compatible with semver.

One possible solution is to require an explicit `const` in the derive:

```rust
#[derive(const Clone)]
pub struct Foo(Bar);

struct Bar;

impl const Clone for Bar {
    fn clone(&self) -> Self { Bar }
}
```

which would generate a `impl const Clone for Foo` block which would fail to compile if any of `Foo`'s
fields (so just `Bar` in this example) are not implementing `Clone` via `impl const`. The obligation is
now on the crate author to keep the public API semver compatible, but they can't accidentally fail to
uphold that obligation by changing private things.

## RPIT (Return position impl trait)

```rust
const fn foo() -> impl Bar { /* code here */ }
```

does not allow us to call any methods on the result of a call to `foo`, if we are in a
const context. It seems like a natural extension to this RFC to allow

```rust
const fn foo() -> impl const Bar { /* code here */ }
```

which requires that the function only returns types with `impl const Bar` blocks.

## Specialization

Impl specialization is still unstable. There should be a separate RFC for declaring how
const impl blocks and specialization interact. For now one may not have both `default`
and `const` modifiers on `impl` blocks.

## `const` trait methods

This RFC does not touch `trait` methods at all, all traits are defined as they would be defined
without `const` functions existing. A future extension could allow

```rust
trait Foo {
    const fn a() -> i32;
    fn b() -> i32;
}
```

Where all trait impls *must* provide a `const` function for `a`, allowing

```rust
const fn foo<T: ?const Foo>() -> i32 {
    T::a()
}
```

even though the `?const` modifier explicitly opts out of constness.

### `const` traits

A further extension could be `const trait` declarations, which desugar to all methods being `const`:

```rust
const trait V {
    fn foo(C) -> D;
    fn bar(E) -> F;
}
// ...desugars to...
trait V {
    const fn foo(C) -> D;
    const fn bar(E) -> F;
}
```

## `?const` modifiers in trait methods

This RFC does not touch `trait` methods at all, all traits are defined as they would be defined
without `const` functions existing. A future extension could allow

```rust
trait Foo {
    fn a<T: ?const Bar>() -> i32;
}
```

which does not force `impl const Foo for Type` to now require passing a `T` with an `impl const Bar`
to the `a` method.

## `const` function pointers

```rust
const fn foo(f: fn() -> i32) -> i32 {
    f()
}
```

is currently illegal. While we can change the language to allow this feature, two questions make
themselves known:

1. fn pointers in constants

    ```rust
    const F: fn() -> i32 = ...;
    ```

    is already legal in Rust today, even though the `F` doesn't need to be a `const` function.

2. Opt out bounds are ugly

    I don't think it's either intuitive nor readable to write the following

    ```rust
    const fn foo(f: ?const fn() -> i32) -> i32 {
        // not allowed to call `f` here, because we can't guarantee that it points to a `const fn`
    }
    ```

Thus it seems useful to prefix function pointers to `const` functions with `const`:

```rust
const fn foo(f: const fn() -> i32) -> i32 {
    f()
}
const fn bar(f: fn() -> i32) -> i32 {
    f() // ERROR
}
```

This opens up the curious situation of `const` function pointers in non-const functions:

```rust
fn foo(f: const fn() -> i32) -> i32 {
    f()
}
```

Which is useless except for ensuring some sense of "purity" of the function pointer ensuring that
subsequent calls will only modify global state if passed in via arguments.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Everything has been addressed in the reviews
