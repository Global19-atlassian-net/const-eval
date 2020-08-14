# Constants in patterns

Constants that pass the [structural equality check][struct-eq] can be used in patterns:
```rust
const FOO: i32 = 13;

fn is_foo(x: i32) -> bool {
  match x {
    FOO => true,
    _ => false,
  }
}
```
However, that check has some loopholes, so e.g. `&T` can be used in a pattern no matter the `T`.

The way this works is that the constant is converted to a [THIR pattern](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir_build/thir/pattern/enum.PatKind.html).
This proceeds recursively through fields of the constant until we hit a reference.
Then, `&[u8]` and `&str` are treated specially, while everything else becomes just a `PatKind::Constant`.

For further compilation, `Constant` are translated to [`PartialEq::eq`][compile-partial-eq], whereas everything else is translated as if the constant was inlined as a pattern.
[RFC 1445](https://github.com/rust-lang/rfcs/blob/master/text/1445-restrict-constants-in-patterns.md) lays the groundwork for changing this in the future to always using `PartialEq::eq`, but no decision either way has been made yet.

[struct-eq]: https://github.com/rust-lang/rust/blob/2c28244cf0fc9868f55070e55b8f332d196eaf3f/src/librustc_mir_build/hair/pattern/const_to_pat.rs#L121
[compile-partial-eq]: https://github.com/rust-lang/rust/blob/2c28244cf0fc9868f55070e55b8f332d196eaf3f/src/librustc_mir_build/build/matches/test.rs#L355

## Soundness concerns

Most of the time there are no extra soundness concerns due to const-patterns; except for surprising behavior nothing can go wrong when the structural equality check gives a wrong answer.
However, there is one exception: some constants participate in exhaustiveness checking.
If a pattern is incorrectly considered exhaustive, that leads to a critical soundness bug.

Exhaustiveness checking is done based on the THIR pattern tree, with `PatKind::Constant` being ignored.
This means we can write:
```rust
#![feature(exclusive_range_pattern)]
#![feature(half_open_range_patterns)]
const PAT: &[u8] = &[0];

pub fn test(x: &[u8]) -> bool {
    match x {
        PAT => true,
        &[] => false,
        &[1..] => false,
        &[_, _, ..] => false
    }
}
```

To ensure soundness of exhaustiveness checking, it is crucial that all data considered this check is fully immutable.
In particular, for constants of reference type, it is important that they only point to immutable data.
For this reason, the static const checks reject references to `static` items.
This is a new soundness concern that otherwise does not come up during CTFE.
Note that this is orthogonal to the concern of [*reading from* a mutable `static`](const.md#reading-statics) during const initialization, which is a problem for all `const` (even when they are not used in patterns) because it breaks the "inlining" property.
A more precise check could be possible, but is non-trivial: even an immutable `static` could point to a mutable `static`; that would have to be excluded.
(We could, in principle, also rely on it being UB to have a shared reference to mutable memory, but for now we prefer not to rely on the aliasing model like that---aliasing of global pointers is a tricky subject.)

*Dynamic check.*
To dynamically ensure constants only point to read-only data, we maintain a global invariant:
pointers to a `static` (including memory that was created as part of interning a `static`) may never "leak" to a non-`static`.
All memory outside of `static`s is marked as immutable.
As a consequence, non-`static`s recursively only point to immutable memory.

This check is uncomfortably close to the static checks, and fragile due to its reliance on a global invariant.
Longer-term, it would be better to move to a more local check, by making validation check if consts are recursively read-only.
This check (unlike our current validation) would have to descend through pointers to `static`.

An alternative could be to wait for the [value tree proposal](https://github.com/rust-lang/compiler-team/issues/323) to be implemented.
Then we could check during value tree construction that all memory is read-only (if we use the CTFE machine in `const`-mode, that will happen implicitly), and thus we know for sure we can rely on the value tree to be sound to use for exhaustiveness checking.
How exactly this looks like depends on how we resolve [pattern matching allowing many things that do not fit a value tree](https://github.com/rust-lang/rust/issues/74446#issuecomment-663439899).
However, if only the well-behaved part of the value tree is used for exhaustiveness checking, handling these extra patterns should not interact closely with the soundness concerns discussed here.
