- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Allow the types `[T; N]` and `[T]` to be well-formed for `T: ?Sized`, not just `T: Sized`.

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In addition to slices and arrays of `Sized` types, slices and arrays of unsized types are also allowed.

Arrays of unsized types still have a compile-time known length, but their element type is dynamically sized.

A slice or array of `T` can be unsized into a slice or array of `U` (respectively) if and only if a `T` can be unsized into a `U`; for example, with `T = u8` and `U = dyn Debug`:

```Rust
let arr_u8: &[u8; 4] = &[0, 1, 2, 3];
let slice_u8: &[u8] = x;

// Unsize [T; N] to [U; N] where T can be unsized to U
let arr_dyn: &[dyn Debug; 4] = arr_u8;
// Normal array-to-slice unsizing works for arrays of unsized types.
let slice_dyn_from_arr_dyn: &[dyn Debug] = arr_dyn;
// Unsize [T] to [U] where T can be unsized to U
let slice_dyn_from_slice_u8: &[dyn Debug] = slice_u8;

```

Arrays of unsized types cannot be created directly using `[expr; N]` or `[expr, expr, expr]` syntax; they can only be unsized from arrays of sized types, or fallibly converted from slices of unsized types using `TryFrom<&[T]> for &[T; N]` and similar impls.

Most methods and traits on arrays and slices will support unsized element types, with the common exceptions of those which take an array by-value (such as `array::map`), and slice methods which read or write elements in the slice (such as `slice::sort` and `slice::fill`).



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

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

- The `Pointee::Metadata` of slices of `Sized` types would no longer be the simple `usize`, it would be the slightly more complicated `(usize, ())`.

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

- "Diamond unsizing": e.g. how does one unsize a `[u8; 4]` to a `[dyn Debug]`? Will the compiler do that in one step or does the user have to have an intermediate `[u8]` or `[dyn Debug; 4]`?


- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

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
