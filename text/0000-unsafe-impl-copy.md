- Feature Name: `unsafe_impl_copy`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

`Copy` may be `unsafe`ly `impl`emented for types whose fields are not all `Copy`.

# Motivation
[motivation]: #motivation

`unsafe impl Copy` allows types containing as fields types which do not implement `Copy` to be `Copy`; e.g. `core::ops::Range`, `core::cell::UnsafeCell`.

The two main classes of types for which this would be useful are:

1. Types with interior mutability. `UnsafeCell` does not implement `Copy`, and it may be considered a breaking change in soundness guarantees to have it do so, since currently-sound code may be exposing `&UnsafeCell` in public APIs which would become unsound if `UnsafeCell<T: Copy>: Copy`. Despite this, it can be sound to have single-threaded interior-mutable `Copy` types, for example `Cell<T: Copy>` could soundly implement `Copy` (though that impl is intentionally not included in this RFC; if this RFC is accepted, then the question can be revisited).
2. Types which are not `Copy` only as a lint. `core::ops::Range<Idx: Copy>` and other otherwise-copyable iterators generally do not implement `Copy` as it is considered a footgun to have `T: Iterator + Copy`. However this means that types containing these types are precluded from being `Copy` themselves.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Some types can’t be copied safely. By default, if you try to implement Copy on a struct or enum containing non-Copy data, you will get the error E0204. If you know for sure that your struct or enum can be soundly copied, you can unsafely implement `Copy` with `unsafe impl Copy for MyType {}`.

<!--
Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.
- Discuss how this impacts the ability to read, understand, and maintain Rust code. Code is read and modified far more often than written; will the proposed feature make code easier to maintain?

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.
-->

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

<!--
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

-->

It is a compile-time error to have `impl Drop` and `impl Copy` for the same type, even with `unsafe impl Copy`.

Drop glue of a type with an `unsafe impl Copy` is removed, similarly to as if the fields with drop glue were wrapped in `ManuallyDrop`.

Non-`Copy` fields of a type unsafely implementing `Copy` behave the same as `Copy` fields with respect to moves/copies, i.e. when a place expression that is a (possibly nested) field-projection from a `Copy` type is evaluated in a value expression context, the value will be copied, not moved. For example:

```rust
#[derive(Clone)]
struct CopyRange {
    range: std::ops::Range<usize>,
}
unsafe impl Copy for CopyRange {}

let value = CopyRange { range: 0..4 };
let range1 = value.range;
let range2 = value.range; // No error, since `CopyRange: Copy` and `value.range` is a place-expression
```

The above "exception" only applies to (possibly nested) field-projections. For example:


```rust
use std::ops::Range;
#[derive(Clone)]
struct UnsoundCopy {
    pair: (Range<usize>, Range<usize>),
    many: Vec<std::ops::Range<usize>>,
}
unsafe impl Copy for UnsoundCopy {}

let value = UnsoundCopy { range: 0..4 };
let a = value.pair;
let b = value.pair; // No error, since `value.ranges` is a place expression that is a field-projection from `value`.
let c = value.pair.0;
let c = value.pair.0; // No error, since `value.pair.0` is a place expression that is a (nested) field-projection from `value`.
let c = value.ranges[0] // Error: cannot move out of index of Vec<Range<usize>>, because `value.ranges[0]` is not just a field-expression from `value`
```


# Drawbacks
[drawbacks]: #drawbacks

Some code may assume that `T: Copy` implies that `T` has no interior mutability. Since there are currently no interior-mutable types that implement `Copy` because `UnsafeCell` (the root of all interior mutability in Rust) does not implement `Copy`, this assumption is not entirely unfounded. Allowing `unsafe impl Copy` on types containing `UnsafeCell` would violate this assumption.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!--

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

-->

- This is a language proposal, and cannot in general be entirely replicated in a library (on stable or using existing nightly features)
	+ To wrap a specific type `T`, you could make a `#[repr(C, align(T_align))]` struct holding an appropriately-sized array of `MaybeUninit<u8>` which could implement `Copy` and have by-reference and by-value conversions to/from `T`, but this does not work if `T` contains interior mutability, as `UnsafeCell` is not `Copy`, and cannot be generic, since there is no way to have the alignment of `T` without holding a `[T; 0]` (which is not always `Copy`), and you can only make the array length `size_of::<T>()` on nightly with `#![feature(generic_const_exprs)]`.
- Make `UnsafeCell<T: Copy>: Copy`.
	+ This would alleviate motivation point 1, but would have soundness implications for APIs which expose `&UnsafeCell` publicly (see Motivation), and may be a footgun for unsafe code which did not intend to implicitly copy an `UnsafeCell`.
- Introduce an `ExplicitCopy` trait.
	+ This trait would be for types listed in motivation point 2, for which a memcpy is sound but it would be a footgun to copy implicitly.
	+ (TODO: expand this?)
- Don't make field access of `Copy` type "special"; moving out of a field is a copy or move based only on the type of the field, not the type of the field-projected container.
	+ This may be harder to implement(?) since it may require being able to mark a `Copy` type as partially-moved-out-of when that is not currently possible(?). (or it may be easier to implement, I don't know about the compiler internals regarding moves/copies)
- Have a `TrivialDrop` unimplementable auto-trait implemented by types with no drop glue (including `ManuallyDrop`), and only allow `unsafe impl Copy` for `TrivialDrop` types.
	+ This could be less of a footgun, as it would require users who *really* want to copy structs holding `Drop` types to additionally wrap the fields in `ManuallyDrop`.
	+ This might also be a semver hazard (adding a `Drop` impl to a type could make downstream code which `unsafe impl Copy` for `struct Wrapper(Upstream);` stop compiling), but this could be resolved by considering the downstream user's `unsafe` code to be at fault for assuming something about the upstream code that ended up not being guaranteed, similar to how downstream `transmute`s can fail to compile if the upstream changes the size of a type.
- Re-use the same logic for what types are valid as `union` fields, i.e. only allow `Drop` types when wrapped in `ManuallyDrop`.
	+ This seems to have most of the upsides of `TrivialDrop` without requiring a new auto-trait.

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What types are allowed as fields in types with `unsafe impl Copy`? (Resolve before RFC is merged, or maybe before RFC is proposed and just list the others in alternatives)
	+ Any types (their drop glue is removed as though they were wrapped in `ManuallyDrop`) (current wording)
	+ `TrivialDrop` types (see "Rationale and Alternatives").
	+ Same rules as `union` fields (`Copy` types, references, `ManuallyDrop`, arrays and tuples of such) (I think this is my preference at the moment).
	+ Any types (their drop glue is *not* removed) (I am opposed to this, as it would break the current status quo where `Copy` types have no drop glue).
- Field access semantics (Could be resolved later)
	+ Moving out of a non-`Copy` field of a `Copy` value copies the field (current wording).
	+ Moving out of a non-`Copy` field of a `Copy` value leaves the value partially moved-out-of (like "normal" non-`Copy` fields in non-`Copy` values).
- How does a type indicate that it does not implement `Copy` only as a lint?
	+ Point 2 of the motivation is for field types which could safely implement `Copy` but do not only as a lint.
	+ It is unclear if it would be better for there to be a trait to indicate this, or if the guarantee could just be listed in the type's documentation (and removing such a guarantee from a public type would be a semver-major change).

<!--

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

-->

# Future possibilities
[future-possibilities]: #future-possibilities

- A lint for `unsafe impl Copy for T` if a safe `impl Copy for T` would work.
- A lint for `unsafe impl Copy for T` if `T` contains `&mut` (since copying such a type would likely be immediate UB for non-zero-sized `T`).
- `unsafe impl<T: Copy> Copy for core::cell::Cell<T> {}`
- `unsafe impl<T> Copy for core::mem::MaybeUninit<T> {}`, not just `T: Copy`

<!--

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.

-->
