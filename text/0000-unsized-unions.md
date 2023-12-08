- Feature Name: (`unsized_unions`)
- Start Date: (2023-08-05)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Allow `#[repr(C)]` unions to have at most one unsized type as a field.

# Motivation
[motivation]: #motivation

Allows `MaybeUninit<T>` to support `T: ?Sized`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

I'm not sure how to write a "guide" portion of this that's any simpler than the "reference" portion, given that `union`s are generally only useful in `unsafe` code.

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.
- Discuss how this impacts the ability to read, understand, and maintain Rust code. Code is read and modified far more often than written; will the proposed feature make code easier to maintain?

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Unions may have up to one unsized field. Unions with an unsized field are themselves unsized and their pointer metadata is that of their unsized field.

Unsized unions cannot be directly constructed, they can only be unsized, or accessed using pointers.

The alignment of a value of unsized union type is the maximum of the alignments of it's `Sized` fields and the (possibly dynamic) alignment of its unsized field.

The size of a value of unsized union type is the maximum of the sizes of it's `Sized` fields and the (possibly dynamic) size of its unsized field, rounded up to its alignment.


```rust
#[repr(C)]
union MaybeUninitStr {
	_uninit: (),
	slice: ManuallyDrop<str>,
}
```



This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

### Documentation Changes

`std::marker::Unsize`: Add a section mirroring the existing section on structs, replacing the "only the last field" requirement with an "only one field" requirement.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

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

- `repr(Rust)` and `repr(transparent)` unions.
- Interaction with `?MetaSized` (the unsized field of a union must be `?Sized + MetaSized` so that the size and alignment of a value of the union type can be calculated regardless of if the unsized field is the active field.)

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Safe `enum`s could have similar behavior. Especially in the case of `#[repr(C)]` and/or `#[repr(Int)]` non-C-like enums, which have a defined layout that is defined in terms of `#[repr(C)]` structs and unions which can both (if this RFC is implemented) hold unsized types.
