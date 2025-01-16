- Feature Name: `ptr_metadata_v2`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Add generic APIs that allow manipulating the metadata of wide pointers, as well as creating types with custom wide-pointer metadata:

* Extracting the metadata from a pointer
* Reconstructing a pointer from a data pointer and metadata
* Representing vtables, the metadata for trait objects, as a type with some limited API
* Defining types with custom metadata
* Possibly allowing structs, unions, arrays, slices, etc to contain one or more unsized fields.

# Motivation
[motivation]: #motivation

We currently have two "primitive" unsized types in stable Rust: slices and trait objects. Additionally, `struct`s may have a single
unsized field at the end (the "tail"). Pointers to such unsized types are "wide" and contain metadata describing

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

TODO

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

The APIs whose full definition is found below are added to `core::ptr` and re-exported in `std::ptr`:

* A `Metadata<T>` type (primitive or otherwise magic lang item) holds the metadata of a pointer to `T`, for all types `T`.
* A `SimplePointee` trait which is implemented for types whose metadata consists of a single field, whose type is given as the associated type `<T as SimplePointee>::SimpleMetadata`.
* A `trait Thin = SimplePointee<SimpleMetadata = ()>` trait alias. All `Sized` types are `Thin`, as well as `extern type`s.
* A `WithMetadata<M>` type (primitive or otherwise magic lang item), which is unsized and implements `SimplePointee<SimpleMetadata = M>`
  which can be used to define structs with custom metadata.
* A `metadata` free function.
* A `DynMetadata<Dyn>` struct that is the `SimpleMetadata` of `dyn Trait` trait objects.
* A `from_raw_parts` constructor and `to_raw_parts` method for each of `*const T`, `*mut T`, adn `NonNull<T>`.

The bounds on `null()` and `null_mut()` function in that same module as well as the `NonNull::dangling` constructor are changed
from (implicit) `T: Sized` to `T: ?Sized + Thin`. Similarly for the `U` type parameter of `NonNull` and pointers' `cast` methods.
This enables using those functions with `extern type`s.

The `SimplePointee` trait is implemented for most types. Pointers can be `as` casted between if their pointees both
implement `SimplePointee` with the same `SimpleMetadata`, or if the output pointee implements
`Thin` (a.k.a. `SimplePointee<SimpleMetadata = ()>`).

```rust
pub type Metadata<T: ?Sized> = /* compiler magic */;

impl<T: ?Sized> Debug + Copy + Send + Sync + Ord + Hash + Unpin + Freeze for Metadata<T> {}

pub type WithMetadata<M: Debug + Copy + Send + Sync + Ord + Hash + Unpin + Freeze> = /* compiler magic */;

pub trait SimplePointee {
    type SimpleMetadata: Debug + Copy + Send + Sync + Ord + Hash + Unpin + Freeze;
}

pub trait Thin = SimplePointee<SimpleMetadata = ()>;
```

## `Metadata<T>`

`Metadata` is a "magic" type that has different fields depending on its generic parameter:

### Primitive Thin Pointees

For all `T` = integers, floats, `bool`, `char`, `!`, pointers, references, `fn` pointers, `fn` items, closures, `extern type`s,
or `Metadata<U>` itself, `Metadata<T>` is a fieldless 1-ZST struct.

### Arrays and Slices

* For `str`, `Metadata<str>` has a single field `pub length: usize`.
* For `[T]`, `Metadata<[T]>` has two fields: `pub length: usize`, and `pub element: Metadata<T>`.
* For `[T; N]`, `Metadata<[T; N]>` has a single field `pub element: Metadata<T>`.

### Trait objects:

For `dyn Trait`, `Metadata<dyn Trait>` has a single field `pub vtable: DynMetadata<dyn Trait>`

### Structs and Unions

When `T` is a `struct` or a `union`, `Metadata<T>` has fields with the same names and visibilties as `T` does, whose types
are `Metadata<F>` where `F` is the corresponding field's type.

For example:

```
pub struct Foo<T: ?Sized> {
	x: u32,
	pub(crate) y: u32,
	pub z: T
}
// expository
pub struct FooMetadata<T: ?Sized> {
	x: Metadata<u32>,
	pub(crate) y: Metadata<u32>,
	pub z: Metadata<T>,
}

pub struct Bar<T: ?Sized>(pub u32, T);
// expository
pub struct BarMetadata<T: ?Sized>(pub Metadata<u32>, Metadata<T>);
}
```

The visibilities are relative to `T`'s definition. Likewise, if `T` is `#[non_exhaustive]`, then so is `Metadata<T>`
(again relative to `T`'s definition).

### Tuples

Tuples are essentially treated as tuple-structs with all-public fields.

For example:

```rust
type Tuple = (u32, f64, [u8]);
// expository
struct TupleMetadata(pub Metadata<u32>, pub Metadata<f64>, pub Metadata<[u8]>);
```

### Enums

TODO

For example:

```rust  
enum Foo {
	A { x: u32, z: [u8] },
	B(dyn Trait),
	C
}
struct FooMetadata {
	A.x: Metadata<u32>,
	A.z: Metadata<[u8]>,
	B.0: Metadata<dyn Trait>,
}
```

### `WithMetadata<M>`

`core::ptr::WithMetadata<M>` is a "magic" type such that `Metadata<WithMetadata<M>>` has a single, public field `metadata: M`.

`WithMetadata<M>` is covariant in `M`.

`WithMetadata<M>` is `!Sized + !MetaSized`. Types containing it as a field must manually implement `MetaSized`.

TODO: alternately, `WithMetadata` is just an `extern type` and having it as a field requires doing whatever that would require.

TODO: alternately, `extern type`s let you specify their metadata and layout, so we don't even need `WithMetadata` as a stdlib type.

TODO

## Trivial, Simple, and Complex Metadata

Currently in Rust, one can cast from `*const [u8]` to ``*const StructWithTail<[u32]>`, e.g., so some way to specify the semantics
of such casts is required.

### Simple Pointees and Simple Metadata

For most unsized types, only one "kind" of metadata is needed: the length of a slice, the vtable of a trait object, etc.

These types are considered "simple pointees", and the single "kind" of metadata they require is their "simple metadata".

Concretely, `SimplePointee` is a trait automatically implemented by the compiler for all appropriate types:

```rust
pub trait SimplePointee {
    type SimpleMetadata: Debug + Copy + Send + Sync + Ord + Hash + Unpin + Freeze;
}

pub trait Thin = SimplePointee<SimpleMetadata = ()>;
```

All `Sized` types implement `Thin` (a.k.a. `SimplePointee<SimpleMetadata = ()>`).

`str` and slices of `Sized` types implement `SimplePointee<SimpleMetadata = usize>`

For `struct`s and `union`s, if all fields but the last implement `Thin`, and the last field implements `SimplePointee`, then the
struct/union implements `SimplePointee<SimpleMetadata = <LastFieldTy as SimplePointee>::SimpleMetadata>`. Fieldless structs/unions
implement `Thin`, as they are `Sized`


### Pointer casting

A pointer `as`-cast from `*const T` to `*const U` is valid if either of the following is true:

* `U: Thin`, i.e. any pointer can be cast to a thin pointer, OR
* `T: SimplePointee` and `U: SimplePointee<SimpleMetadata = T::SimpleMetadata>`, i.e. pointers can be cast to pointers with the
  same simple metadata.

TODO: the above doesn't fully work, if `F: !SimpleMetadata`, then you can't cast from `*const F` to `*const WithTail<F>`,
which currently works under `F: ?Sized`

* OR, TODO: allow if `T` is the tail type of `U` or vice versa, or they have the same tail type.

TODO

### Constructing `Metadata`

`Metadata` can only(?) be constructed with a braced struct expression (like `Metadata { length: 42 }`).

When constructing a `Metadata` for most struct types, it is expected that most fields will be trivial. Therefore, similar to
[RFC 3681](https://github.com/rust-lang/rfcs/blob/master/text/3681-default-field-values.md), not all fields of `Metadata`
are necessary to explicitly write. Any field whose type is `Metadata<F>` where `F: Thin` can be omitted if the struct initilizer
contains `..`.

For example:

```rust
struct Foo {
    x: u32,
    y: [u8],
}

// `m.x` is implicitly given its default (and only possible) value,
// as well as `m.y.element`.
let m = Metadata::<Foo> { y: Metadata { length: 42, .. }, .. };
```

TODO: For `Metadata` specifically, should the `..` be required?

TODO: Interaction of trivial defaulting with `#[non_exhaustive]`

TODO

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

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
