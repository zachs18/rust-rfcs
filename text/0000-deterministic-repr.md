- Feature Name: `repr_deterministic`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

A new layout strategy that is guaranteed to be deterministic in one (codegen unit/compilation/something else?) based only on the size and alignment of the type's fields.

# Motivation
[motivation]: #motivation

It is sometimes necessary to have a fixed layout for a type, e.g. to allow transmuting between types intended to have the same layout. However, currently the only way to do this is with `repr(C)`, which
* prescribes a fixed layout for the type, leaving `rustc`'s layout optimization potential on the table
* may require a change to declaration order (and thus drop order) of fields to achieve minimal size.

For a concrete example, when implementing a tree-map data structure, the general layout of a node will 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

`repr(deterministic)`

This `repr` allows the compiler to make some layout optimizations, but requires that the optimizations it makes are deterministic within a compilation based only on the size, alignment, and declaration order of fields and variants, any other `repr` annotations on the type.

This allows a limited but useful number of transmutations to be sound.

The only `repr`s that `repr(deterministic)` is allowed to be combined with are `repr(packed)` and `repr(align)`. Concretely,

* `repr(transparent)`, `repr(C)`, `repr(Int)` (on an enum), or `repr(C, Int)` (on an enum) may not be used with `repr(deterministic)`, as they already fully define the layout of a type.

## `repr(align)` and `repr(packed)`

For two `repr(deterministic)` types to have the same deterministic layout, they must have the same `repr(align(N))` or `repr(packed(N))` annotation (or lack thereof), in addition to the other requirements. Otherwise, `repr(packed(N))` and `repr(align(N))` behave the same on `repr(deterministic)` types as they do on other types.


## `repr(deterministic)` on `struct`s

When a struct is annotated with only `repr(deterministic)`, then the struct will have a layout that is wholly determined by the size and alignment of its fields, for example

```rs
#[repr(deterministic)]
struct SignedInts {
	a: i32,
	b: i64,
	c: i8,
}

#[repr(deterministic)]
struct UnsignedInts {
	d: u32,
	e: u64,
	f: u8,
}
```

It is sound to `transmute` between `SignedInts` and `UnsignedInts`, since they have the same number of fields, and their fields, compared pairwise in declaration order, have the same sizes and alignments.

**Note**: Unline `repr(C)`, the specific layout of the type *is not defined*; for example, `a` could be at offset `0`, or `8` (or something else) in `SignedInts`, but whatever offset it is at, `d` is at the same offset in `UnsignedInts`.

## `repr(deterministic)` on `enum`s

For two enums to be have the same deterministic layout, they must have the same number of variants, and each variant in the two enums must, copmpaired pairwise in declaration order:
* ~~Have the same explicit discriminant (or have no explicit discriminant)~~ (explicit discriminants are only allowed with `repr(Int)`?)
* Either both be unit variants, or both be struct variants obeying the `repr(deterministic)` rules for `struct`s.

In addition to "raw layout" (i.e. field offsets, size, alignment), `repr(deterministic)` on an enum also guarantees that the discriminant values are the same between two enums with the same deterministic layout, such that transmuting one to another will give the corresponding variant

For example,

```rs
#[repr(deterministic)]
enum SignedInt {
	I64(i64),
	I32(i32),
	I8(i8),
}

#[repr(deterministic)]
enum UnsignedInt {
	U64 { value: u64 },
	U32 { value: u32 },
	U8 { value: u8 },
}
```

`SignedInt::I64(42)` can be soundly transmuted to `UnsignedInt` and will give `UnsignedInt::U64 { value: 42 }`. Likewise, `SignedInt::I8(-1)` can be soundly transmuted to `UnsignedInt` and will give `UnsignedInt::U8 { value: 255 }`.
<!-- 
---
Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.
- Discuss how this impacts the ability to read, understand, and maintain Rust code. Code is read and modified far more often than written; will the proposed feature make code easier to maintain?

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms. -->

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

(Added to the "Representations" section of the "Type Layout" pageof the reference)

### The `deterministic` Representation

The `deterministic` representation can be used on a `struct`, `enum`, or `union`.

Types with this representation have an unspecified but deterministic layout that depends only on:

* type kind (`struct`/`enum`/`union`)
* the compilation target and compiler version and configuration
* the size, alignment, and declaration order of variants and fields
* the presence of a `?Sized` generic tail field
* the `packed` or `align` representation (if appplicable)

such that two types with this representation with the same such determiners have the same layout, field offsets, and discriminant values.

Notably, things which do *not* affect the layout of such a type include:

* whether the struct or a variant of the enum is a tuple-struct or named-field struct
* the names of fields
* the names of variants.

If a type with the `deterministic` representation is unsizeable, then field offsets of the non-unsized fields do not depend on the unsized field's type's size or alignment (otherwise unsizing would not work).



---

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

* This would make layout computation in the compiler more complicated, and could introduce bugs if TODO


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Users can use `repr(C)` and deal with possible missed optimizations and/or manually layout fields optimally.
- Users can have substructs that are `repr(Rust)` and outer structs that are `repr(C)` to enforce only some layout requirements

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

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

* 1-ZSTs fields could be ignored for the layout calculation; and structs/variants with no non-1-ZST fields could be considered compatible with `()`/a unit variant.
* `repr(deterministic)` could be allowed to be combined with `repr(Int)` on enums (repr must be the same, explicit discriminants must be the same)
* Single-variant `repr(deterministic)` enums could be considered compatible with `repr(deterministic)` structs, if the variant declared as a struct would be compatible.
* Could relax the "field declaration order" requirement for unions, but that would require that same-layout fields are always at the same offset.

# Future possibilities
[future-possibilities]: #future-possibilities



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
