- Feature Names: tracked_values, allocators
- Start Date: 2015-09-21
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Important Note

There are various changes draft RFC pending for this RFC. I'm keeping
a list of the [pending changes][], embedded in the text, right after
the [summary][] section.

# Summary
[summary]: #summary

One sentence summary: Provide a standard user-defined allocator API
and lay the ground work for garbage collection.

To achieve this, this RFC proposes three changes to Rust:

 1. type-based value tracking,

 2. add tracked allocation methods to the `#[allocator]` crate API, and

 3. a *pair* of user-implementable `Allocator` traits that
    each respectively support or renounce value tracking.

The main user-visible portion of an implementation of this RFC will be
the `Allocator` traits from item 3 above. Much of this RFC is
concerned with how to support garbage collection, and such support
constrains the design space of the Allocator API; that is why the two
topics are presented together here.

# Pending Changes
[pending changes]: #pending-changes

This is a list of pending changes. I'm writing it up here so that people
with a link to this draft don't overlook it.

 * [ ] Add (potentially in an appendix) a high-level introduction to
       GC and the issues with integrating third-party GC with Rust
       code. Current text deep-dives too quickly.

 * [ ] Must be able to disable tracking globally by swapping in
       appropriate `#[allocator]`, and allow libraries to observe this
       (so they can avoid building up unused metadata).

 * [ ] The [linting against misclassification][] approach is not sound
       on its own. Switch to an actually sound approach (such as
       casting restrictions discussed in that section).

 * [ ] Add alternative of removing the type-based
       `Tracked`/`Untracked` distinction and "just" tracking all
       allocations when GC is enabled. (Relies on disabling tracking
       globally via `#[allocator]` swapping to retain any hope of
       claiming "zero-cost".)

 * [ ] Need to specify the default standard-library allocator(s),
       e.g. at least one should implement `TrackingAllocator`.

 * [ ] Need actual code examples of a tracking user-defined `Allocator`,
       preferably one that illustrates how to set up `BDFunc`.

 * [ ] Need discussion of concurrent allocator access. Namely, need to
       point out that the metadata construction, introspective access,
       and maintenance within the tracking `#[allocator]` extensions
       (and `BDFunc` too) must be thread-safe. (Of course one could
       accomplish this via a global lock; the point is just to note
       that it is a requirement.)

# Table of Contents
* [Summary][summary]
* [Motivation][motivation]
* [Detailed Design][detailed design]
* [Drawbacks][drawbacks]
* [Alternatives][alternatives]
* [Unresolved Questions][unresolved questions]
* [Appendices][appendices]

# Motivation
[motivation]: #motivation

The main motivation for this RFC is to allow user-defined allocators.

The RFC covers more terrain than just the definition an `Allocator`
trait, because this RFC seeks to define the lowest level primitives
that user code will need to use in order to define an allocator that
integrates with third-party garbage collectors.

## Allocators
[allocators]: #allocators

As noted in [RFC PR 39][], modern general purpose allocators are good,
but due to the design tradeoffs they must make, cannot be optimal in
all contexts.  The standard library should allow clients to plug in
their own allocator for managing memory.

The typical reasons given for use of custom allocators in C++ are among the
following goals:

  1. Speed: A custom allocator can be tailored to the particular
     memory usage profiles of one client.  This can yield advantages
     such as:
     * A bump-pointer based allocator, when available, is faster
       than calling `malloc`.
     * Adding memory padding can reduce/eliminate false sharing of
       cache lines.

  2. Stability: By segregating different sub-allocators and imposing
     hard memory limits upon them, one has a better chance of handling
     out-of-memory conditions.  If everything comes from a global
     heap, it becomes much harder to handle out-of-memory conditions
     because the handler is almost certainly going to be unable to
     allocate any memory for its own work.

  3. Instrumentation and debugging: One can swap in a custom allocator
     that collects data such as number of allocations or time required
     to service requests.

[RFC 1183][] provides clients with the ability to swap in their own
allocators *globally*; this capability may satisfy goal 1 or 3 above
(it cannot satisfy goal 2). [RFC 1183][] itself even lists goal 3 as a
secondary motivation, but its primary motivation was interoperability
when embedding Rust into other runtimes.

However, the addition of the `Allocator` traits proposed here is based
on the hypothesis that replacing the allocator globally throughout the
application is too inflexible; i.e. programmers want the ability to
feed different allocators into specific object factories or containers
within a library. All three goals given above can be applied to
particular software components rather than an application as a whole.

## GC support
[GC support]: #gc-support

We want to allow Rust applications and libraries to integrate with
native libraries that use tracing Garbage Collection (GC) or other
automatic memory management schemes that require tracing through an
object graph.

For example, this is necessary when one wants a crate that
interoperates with a third-party javascript engine (such as V8 or
SpiderMonkey) and allows a root reference to a value managed by the JS
engine's GC to flow into the Rust code.

This RFC on its own does not provide all the details required for full
integration with third-party GC's (such as the LLVM changes for 
generating stack maps). The goal here is not to define such
GC support; it is instead to define API's that will *allow* such
support to be added in the future. See [GC details out of scope][] for
more discussion.

## Scope of this RFC
[scope of this RFC]: #scope-of-this-rfc

Note: With respect to GC, this RFC is solely concerned with the issues
regarding the reflective mechanisms needed to identify roots, and the
constraints such mechanisms impose on allocator designs.

In particular, the definition of a "nice" `Gc<T>` type with
appropriate ergonomics for use in implementing a Rust crate, and any
traits for user-defined tracing, is *out of scope* for this RFC.

In addition, as noted above, some details of how a third-party GC
library would integrate with Rust have been left unspecified here; see
further discussion in the [GC details out of scope][] section.

Root identification is the GC-related goal here, not the remainder of
garbage collection; that is why the examples use a hypothetical
`HasRoots` type rather than the `Gc<T>` that one might otherwise
expect there.

The overall goal is that users can start writing their own containers
parameterized over an appropriate `Allocator` trait, or their own
implementation of one of the two `Allocator` traits, and that the code
they write today will continue to work as GC support is added in the
future.

# Detailed design
[detailed design]: #detailed-design

We first present the high-level overview of the design's four
components, then the foundations for why this design why chosen, and
then dive into the details of each component in turn.

## Overview

(Each component links to its details section for more info.)

* [Value Tracking][value tracking]: Add a marker trait, `Tracked`,
  that indicates that such values must be identifiable when embedded
  into owned objects transitively reachable from the stack. As already
  mentioned, the tracking performed via this trait is to identify GC
  roots, not arbitrary GC managed data.

* [Classification][classification]: Add traits and functions to allow
  static and dynamic classification of types and values into
  "maybe-tracked" and "definitely-untracked".

* [High-level Allocator traits][high-level allocator traits]: Add
  traits that client code implements to provide their own allocation
  procedures, and are used as bounds in allocator-parametric
  containers. This RFC proposes adding two such allocator traits: one
  trait that *all* allocators implement, and a second trait used to
  mark allocators that *ensure* support for value tracking.

* [Low-level `#[allocator]` API][low-level allocator api]: Add
  low-level allocator methods to support mapping addresses to a
  client-supplied function for inspecting tracked blocks of
  memory. These functions are meant solely for the implementation of
  value-tracking allocators

Those above components cover the four main additions
described by this RFC.

Before diving into the details of what each component provides, we now
discuss why GC integration requires such machinery. You may wish
to skip ahead to the [actual changes proposed by the RFC][] section
if you are already familiar with the issues of integrating a GC
into Rust, since the foundations section is lengthy.

## Foundations

### Capabilities Needed for GC Integration
[Capabilities]: #capabilities-needed-for-gc-integration

Integrating with a GC requires running code be capable of identifying
whether certain "root" values are still actively in use.

In a runtime where *all* heap-allocated values are GC-managed, this
would be handled by treating the processor registers and the stack as
*the* source of roots. In the Rust runtime, we must generalize this
idea slightly, by allowing explicitly-managed objects to also hold
roots.  (Otherwise, we would not be able to compose types like
`Vec<HasRoots>`, which is represented by allocating an array of
`HasRoots` values on the explicitly-managed global heap.)

In other words, running code can inspect its own state and identify
all references to gc-managed values reachable from the stack or from
manually-managed objects that it owns (i.e., owned by its stack).

 * Some GC libraries may need to generalize such inspection to include
   objects owned by the stacks of other threads. (For example, a GC
   libraries where the roots are `Send`). This generalization might be
   a natural extension, but it is not an explicit objective of this
   RFC.

We would like such state inspection to be informed by the static type
system; in other words, it would be nice if the types assigned to
local variables could drive the process of finding roots, so that
the Rust compiler would generate code focused solely on variables
with types like `HasRoots` or `Vec<HasRoots>`, while types like
`*const u8` or `u64` would be ignored by the scanning process.

Unfortunately, Rust considers it formally safe to freely cast native
pointers via `as` in a manner that discards their underlying type.
For example, one can take a `&HasRoots` and cast it via `as` to
`*const HasRoots`, and then that value can be casted to `*const U` for
any type `U`.

(One can also just cast the pointer to a `usize` and back again to a
`*const U`; this RFC regards such a cast, especially if used as the
*sole* owner for an object, as likely to foil GC integration, unless
it uses a conservative scheme for root discovery; see further
discussion in the [how will value-tracking be used][] section.)

This is all possible without the use of `unsafe` -- an `unsafe` block
is only required for a dereference, not the casts back-and-forth.

This shows it is easy to construct examples where a local variable
that owns roots does not have a type that reflects such ownership, and
thus it would be quite brittle rely on the assigned types alone to
drive the root-identification process.

 * One could attempt to work around this by requiring all root data to
   never leak into such a context, but I do not expect this to be
   feasible, since so much of Rust is designed operations that take
   `&T` or `&mut T` and then casts it to native pointers.

Instead, in order to retain interoperability with code that does
perform such casting to native addresses, we will require a degree of
dynamic reflection as runtime, where for any memory address, we can
perform the following queries:

 1. Is this address part of a memory that may own roots?

 2. If so, what is the starting address of this block of memory?

 3. Finally, how do I scan this block of memory to discover other
    addresses of potential interest?

This RFC proposes supporting these three queries by adding support for
"tracked memory blocks".

### How will value-tracking be used
[how will value-tracking be used]: #how-will-value-tracking-be-used

"How will value-tracking be used?" is a question that is up to the GC
implementation.  (See also the [GC details out of scope][] section.) It is
not directly related to this RFC, except that it gives some motivation
for *why* the functionality is being provided at all.

Hypothetically, in a language and runtime with perfect type
information and appropriate compiler integration, a GC would not need
such dynamic value-tracking at all.  It would "just" use
statically-generated stack maps to determine the locations and types
for each word of interest on the stack.

 * Such a system *could* even handle `Box<Trait>`, assuming it
   is represented in the same uniform DST form, by inspecting the
   vtable to find the underlying type of the referenced data.

Unfortunately, in Rust user-written code may subvert the type system,
as described in the earlier [capabilities section][Capabilities].
While we could say that such code is simply "incorrect", I suspect the
difficulty in actually identifying all such code would be too hard for
any real-world client seeking GC integration.

Therefore this RFC proposes a different solution: require allocation
of tracked values to include sufficient reflective functionality to
extract the information embedded within them.

A GC would be able to use these reflective capabilities when looking
up addresses on the stack.

It is possible that would suffice, and once we are scanning values
embedded into heap-allocated blocks, it may be reasoanble to trust the
type-information of each value to guide a more precise scan of
subsequent blocks that it uncovers during its search for roots.

Or it may be that a GC will need to use value-reflection recursively,
(that is, even when looking at addresses that are not on the
stack). As it scans the value embedded within a block, the GC might
need to continue using the block descriptors for each tracked address
that it finds, in order to uncover all of the roots.

(These two fairly different perspectives reflect my own unsureness
about how much we will be able to trust the type information for an
arbitrary Rust program in order to scan the blocks it allocates.)

Note that the RFC as written allows for one to ask if any address is
tracked. This should even allow for a fairly conservative root
scanning scheme that scans the blocks without much consideration of
the static types of the values embedded within it, such as woud be
used by a Bartlett-style mostly-copying collector.

 * Such conservativeness is not *required by* the design -- merely
   *supported*. We are leaving as many options open for GC as we
   reasonably can.

### GC details out of scope
[GC details out of scope]: #gc-details-out-of-scope

This RFC is solely concerned with the issues regarding the reflective
mechanisms one would need to scan the graph of manually-managed
objects owned by the stack (in order to identify the set of roots that
are used to initiate a collection cycle).

In particular, the definition of a "nice" `Gc<T>` type with
appropriate ergonomics for use in implementing a Rust crate (such as
implementing the `Deref` trait on such `Gc<T>` types) is *out of
scope* for this RFC.

The level of GC integration envisaged by this RFC is one where either:

 * the roots live *solely* as values embedded within blocks reachable
   from the stack, and the programs must ensure that all garbage
   collector actions never need to rewrite any value that has been
   stored on the stack, or

 * the `Tracked` values are more deeply integrated with the compiler,
   to ensure that LLVM can always report what the roots are,
   and rewrite them in reponse to collector actions.

This RFC takes no position as to which of the above kind of
integration we may see in the future. It merely defines what API an
allocator must obey to allow root scanning in the future.

### Root identification: Similar to Tracing, but not the same
[similar to tracing]: #root-identification-similar-to-tracing-but-not-the-same

The allocator protocol in this RFC attempts to be orthogonal to the
tracing procedure employed by whatever GC is linked in.

This process of scanning blocks of memory for its roots seems similar,
but need not be identical, to the tracing process performed by the GC.

For example, some GC's will use a stop-and-copy procedure where each
object is moved to a new location when it is first identified as live.
Other GC's may only move some objects, or move no objects at all.

The Root identification procedure is solely scanning the owned memory
to find roots. It does not move nor compact the owned memory.

Also, our inspection of addresses held on the stack (to determine what
blocks of memory the stack owns) may seem similar to a Bartlett-style
"mostly-copying" collector. A crucial difference, however, is this: in
the [model][Model for GC Integration] we are using, the addresses held
on the stack are pointing to explicitly managed memory.

 * Any object that we find via this technique is live, in the sense
   that its containing block has not been freed by the low-level
   allocator, and its high-level allocator has confirmed that it also
   considers the value to be live.

 * Compare this to a common complaint about conservative pointer
   identification schemes, where the chance of false positive of an
   address left over on the stack implies that a GC-managed object
   (and anything it can reach) will be retained, even if the object
   cannot be reached by via live variables on the stack.

 * The latter objection does not apply to what we are doing, because
   the root-containing objects we are identifying are *not*
   GC-managed, and thus any matching address represents an object that
   has not been freed.  We are leveraging the property that all roots
   live within explicitly-managed memory.

 * (One might reasonably ask why we "simply" don't traverse *all* of
   the explicitly-managed tracked memory, rather than scanning the
   stack. The reason is that some of the explicitly-managed memory may
   be owned solely via GC-managed objects, and may in fact form a
   cycle with otherwise unreachable GC-managed objects.  The
   stack-scanning technique is our way of filtering *out* those false
   roots.)

## Actual changes proposed by the RFC
[actual changes proposed by the RFC]: #actual-changes-proposed-by-the-rfc

### Value Tracking
[value tracking]: #value-tracking

Add marker trait, `Tracked`, that a struct/enum can implement to
opt-in to being "tracked", meaning that values of such a type are
identified when embedded into owned objects reachable from the stack
(potentially indirectly reachable).

A tracked value is either: a value of a type that implements
`Tracked`, or a value that "owns" a tracked value.
By "owns", we mean that it either has a field of some type that
implements `Tracked`, or it references a tracked value via
a `*const`/`*mut` pointer.

#### Hypothetical future use of `Tracked`

The intention is that, when we add more GC support, we can, for
example, add an intrinsic, `tracked_addresses`, that for *any*
`T:Sized`, maps a value of type `*const T` to the collection of
(potentially) tracked fields embedded within the referenced `T`.

 *  The actual signature of `tracked_addresses` is:

    ```rust
    pub type Cursor = usize;
    fn tracked_addresses<T>(*const T, Cursor) -> Some(*const T, TypeId, Cursor);
    ```

    One is meant to use a protocol of iteratively invoking the intrinsic,
    starting with `cursor == 0` and then feeding in the returned cursor

 * This intrinsic takes a dynamic value as input rather than a static
   type in order to handle enums, where the number and location of the
   tracked fields depends on runtime properties of the vaue.

### Classification
[classification]: #classification

Add an opt-in built-in trait (OIBIT), `trait Untracked { }`.

 * This allows static classification of a type as "untracked."

 * Add a default impl of `Untracked` for all types: `impl Untracked for .. { }`,

 * Add a negative impl of `Untracked` for any type that implements
  `Tracked`: `impl<T:Tracked> !Untracked for T { }`

 * Note: the above negative impl does not actually work with the
   current OIBIT system.  Nonetheless, the *idea* is the important
   thing: An `impl` of `Tracked` for `T` will silently insert a
   negative impl of `Untracked` for `T`.
   
 * If necessary, make `Untracked` a lang-item to accomplish the latter
   point. We will probably need to make `Untracked` a lang-item for
   other reasons; see [tracking and trait objects][] section.

Add an intrinsic, `is_tracked::<T>() -> bool`, that returns false only
if all values of type `T` can never have tracked values embedded
within themselves or any values they own.

#### Linting against misclassification
[linting against misclassification]: #linting-against-misclassification

Add a lint that triggers on any cast or coercion that converts a
pointer to a `Tracked` type to a pointer to a type that is not
tracked.

 * This is meant to reduce the chance of people from inadvertantly
   losing pointers that need to be tracked, e.g. by storing a `*mut
   HasRoots` into a location of type `*mut u8`, which could be held in
   untracked memory.

 * Originally I wanted to put something stronger than a lint here,
   such as making such casts `unsafe`, but that would break
   parametricity properties of the language; e.g. you cannot put in a
   rule like that without getting arguaby false positives in cases
   like:

   ```rust
   impl<S, T> fn foo(s: *mut S, t: T) {  ... s as *mut T ... }
   ```

   (Of course, those same parametricity properties imply that our
   linting analysis is not strong enough to catch all such attempts to
   subvert value-tracking. Trying to extend the lint system to support
   such analysis is out of scope of this RFC.)


### Implementing value tracking

Perhaps the easiest way to start an explanation of how value tracking
can work is with some pictures:

In the simplest case, a tracking (low-level) allocator could satisfy
an allocation request for a single value by asking the OS `malloc` for
a block to hold that one value; in addition, this "simple" tracking
allocator maintains a table mapping address ranges to a "block
descriptor", `BDFunc`, that will tell us what kind of data the block
contains.

```
        +----+        +--------+
        | bd -------> |  *fn -------> BDFunc
        +----+        +        +
     +--------+       |        |
     |        |       | opt    |
 B:  |   v1   |       |        |
     |        |       |   data |
     +--------|       |        |
                      +--------+
```

In the picture above, the `bd` box is located near the block `B`
allocated for `v1`. This reflects that the low-level allocator may use
different strategies to represent the mapping from a block to the
`bd`.

 * Some low-level allocators may use a separate table, as described
   above.

 * Other low-level allocators might use conventions where some blocks
   have their `bd` stored at an address that is computable via
   bit-masking the original address.

 * The main point is that this mapping from address to `bd` is the
   responsibility (aka implementation detail) of the low-level
   allocator; the high-level allocator has *no* knowledege of where
   the `bd` is stored for any given block of memory.

 * However, the value of the `bd` itself *is* the responsibility
   of the high-level allocator.

The block descriptor value (on the right-hand side of the picture)
always contains a function pointer in its first word. That function
(`BDFunc` in the pictures) is responsible for describing the block,
when given the original internal address and a pointer to the block
descriptor.

The block descriptor value may just be a single word, located in
static memory, that just points to the `BDFunc`.

 * This is why the contents beneath the `*fn` in the block descriptor
   value is labelled "opt data."

  * Or the "opt data" may be many words, located in either static
    memory or dynamically allocated memory, carrying other information
    necessary for the `BDFunc` to do its job of describing an
    individual value stored within the block (described in detail
    later).

Each *high-level* allocator may need distinct functions to describe
their respective strategies for laying out values within a block of
memory.

For example, an allocator that lays out multiple values in a single
block of memory will need a different function, denoted `BDFunc'`
below.

```
        +----+        +--------+
        | bd -------> |  *fn -------> BDFunc'
        +----+        +        +
     +--------+       |        |
     |        |       | opt    |
     |   v1   |       |        |
     |        |       |   data |
 B': |   v2   |       |        |
     |        |       +--------+
     |   v3   |
     |        |
     |  ....  |
     |        |
     +--------+
```

Note again: the location of the `bd` value (and how that location is
derived from the addresses of the block `B'`) is decided by the
low-level allocator; but the content of that location is determined by
the high-level allocator when it allocates the block.

In addition, the high-level allocator could actually store the block
descriptor value in the allocated block itself. This again is the
choice of the high-level allocator, as long as it can satisfy the
other requirements of the allocator API with such a structure.

```
        +----+
 +------- bd |
 |      +----+
 |   +--------+
 +-> |  *fn -------> BDFunc''
     +        +
     | opt    |
     |        |
     |   data |
     |        |
     +        +
     |        |
     |   v1   |
B'': |        |
     |   v2   |
     |        |
     |   v3   |
     |        |
     |  ....  |
     |        |
     +--------+
 ```

So, that is the big idea of how responsiblity is divided up between
the system-wide low-level allocator and the distinct high-level
allocators.

### Low-level `#[allocator]` API
[low-level allocator api]: #low-level-allocator-api

At the lowest level of the `#[allocator]` crate, we require the
addition of new `_tracked` variants of its methods, namely:

```rust
fn __rust_allocate_tracked(size: usize, align: usize, desc: BlockDescriptor) -> *mut u8;
fn __rust_deallocate_tracked(ptr: *mut u8, old_size: usize, align: usize);
fn __rust_reallocate_tracked(ptr: *mut u8, old_size: usize, size: usize,
                             align: usize, desc: BlockDescriptor) -> *mut u8;
fn __rust_reallocate_inplace_tracked(ptr: *mut u8, old_size: usize, size: usize,
                                     align: usize, desc: BlockDescriptor) -> usize;
fn __rust_usable_size_tracked(size: usize, align: usize) -> usize;
```

The above methods are all analogous to the pre-existing methods of an
`#[allocator]` crate.

There is an extra argument on some of the methods, `BlockDescriptor`,
which points to an object that itself carries a method for inspecting the block:


```rust
/// Returns null if `addr` is not part of tracked memory. Otherwise
/// returns a pointer to the `BlockDescriptor` object (which is
/// guaranteed to live at least as long as the memory associated with
/// `addr`).
///
/// You feed `addr` into the returned object in order to query for
/// more information about it beyond the fact that it points into
/// tracked memory.
fn __rust_tracked_descriptor(addr: *const u8) -> BlockDescriptor;

/// A `BlockDescriptor` points to an object capable of describing some
/// block of memory.
///
/// (The `BlockDescriptor` object may well be nothing more than a
/// pointer to this function, but in general it may carry other state
/// that we need to query in order to parse the block of memory and
/// discover how to scan the value embedded within.)
pub type BlockDescriptor = *const *BDFunc;

/// The core method for reflecting on a value embedded within a block.
///
/// Pass in the `BlockDescriptor` that you extracted this function from.
///
/// PRECONDITION: `addr` must be an address held within a block of
/// memory that is tracked by `bd`.
///
/// (This block of memory is hereby referred to as "the block".)
///
/// Returns `true` if `addr` is the address of memory that is part
/// of some value embedded within the block; in this case,
/// `value_data_out` is initialized with metadata corresponding
/// to that embedded value.
///
/// Returns `false` if `addr` does not correspond to any memory
/// location held in a *value* allocated within the block; i.e. it is
/// an address of reserved storage that has not been handed out.
pub type BDFunc =
    extern "C" fn(bd: BlockDescriptor,
                  addr: *const u8,
                  value_data_out: *mut ValueData) -> bool;

/// Basic metadata about a tracked value.
/// It has a starting address and size.
/// It has a usable `TypeId` if and only if `type_id_valid` is true.
#[repr(C)]
pub struct ValueData {
    start_address: *const u8,
    size_in_bytes: libc::c_uint,
    type_id_valid: bool,
    type_id: TypeId,
    max_tracked_values: libc::c_uint,
    walker: ValueWalker
}

/// On each invocation, attempts to fill the `tracked_address` and
/// `type_id` with the next address and type_id of a tracked value
/// embedded within the value that starts at `start_address`.
///
/// If there was no remaining tracked value embedded within, returns
/// false.
///
/// Otherwise, returns true and updates the cursor so that the next
/// call will know to start after that field.
///
/// Never modifies `start_address`.
pub type ValueWalker = extern "C" fn(cursor: *mut Cursor) -> bool

/// Tracks state of the walk over a value.
///
/// Initialize with `cursor: 0` and the `start_address` provided by
/// the original `ValueData` that provided the `ValueWalker`; the
/// other fields will be initialized by the `ValueWalker` itself.
#[repr(C)]
pub struct Cursor {
    start_address: *const u8,
    tracked_address: *const u8,
    type_id: TypeId,
    cursor: libc::c_uint,
}
```

### High-level `Allocator` traits
[high-level allocator traits]: #high-level-allocator-traits

This RFC proposes essentially one high-level allocator trait, which
provides maximal flexibility to its *implementors*, and then a second
marker trait that increases the flexibility available to the *clients*
of the trait.

Simply put: The two traits are `trait Allocator { ... }` and
`trait TrackedAllocator: Allocator { }`.

 * Any allocator can be used as an `Allocator`, but that trait is only
   guaranteed to support allocating types that implement `Untracked`.

 * The `TrackedAllocator` extension marker trait is meant to be used
   as a bound with on the allocator when working with a type that does
   not satisfy `Untracked`. (The main use case of this are types that
   carry GC roots).

 * For example, we want the standard library `Vec` to be parameterized
   over `Allocator`, but we want clients to be able to pass in an
   instance of `Allocator` as long as `T: Untracked` in
   `Vec<T, A>`.

Libraries can be parameterized over their allocator, so that users can
supply their own memory allocator to library types like `Vec` and
`HashMap` (once those types are updated to be allocator-parametric).

#### The `Allocator` API

First we define a collection of type aliases; much memory allocation
functionality is defined in terms of `usize` (since it is in the end
all about memory addresses), but it is useful to give more meaningful
names to the types corresponding to their intended usage.

```rust
pub type Size = usize;
pub type Capacity = usize;
pub type Alignment = usize;

pub type Address = *mut u8;
```

Next we have some support functionality.

The `Kind` structure is meant to capture all the ways that one builds
up larger memory structures from smaller ones.

```rust
/// Represents the combination of a starting address and
/// a total capacity of the returned block.
pub struct Excess(Address, Capacity);

/// Category for a memory record.
///
/// An instance of `Kind` describes a particular layout of memory.
/// You build a `Kind` up as an input to give to an allocator.
#[derive(Copy, Clone, Debug, PartialEq, Eq)]
pub struct Kind {
    size: Size,
    align: Alignment,
    tracked: bool,
}

fn size_align<T>() -> (usize, usize) {
    (mem::size_of::<T>(), mem::align_of::<T>())
}

// Accessor methods
impl Kind {
    pub fn size(&self) -> usize { self.size }

    pub fn align(&self) -> usize { self.align }

    pub fn requires_tracking(&self) -> bool { self.tracked }
}

// public constructor methods
impl Kind {
    /// Creates a `Kind` describing the record for a single instance of `T`.
    pub fn new<T>() -> Kind {
        let (size, align) = size_align::<T>();
        let tracked = is_tracked::<T>();
        Kind { size: size, align: align, tracked: tracked }
    }

    /// Produces the `Kind` describing a record that could be used to
    /// allocate the backing structure for `T`, which could be a trait
    /// or other unsized type.
    pub unsafe fn for_value<T: ?Sized>(t: &T) -> Kind {
        let tracked = is_tracked_val::<T>(); // XXX relies on Alternative functionality
        Kind { size: mem::size_of_val(t), align: mem::align_of_val(t), tracked: tracked }
    }

    /// Creates a `Kind` describing the record for `self` followed by
    /// `next` with no additional padding between the two. Since no
    /// padding is inserted, the alignment of `next` is irrelevant,
    /// and is not incoporated *at all* into the resulting `Kind`.
    ///
    /// Returns `(k, offset)`, where `k` is kind of the concatenated
    /// record and `offset` is the start of the `next` embedded witnin
    /// the concatenated record (assuming that the record itself
    /// starts at offset 0).
    ///
    /// (The `offset` is always the same as `self.size()`; we use this
    ///  signature out of convenience in matching the signature of
    ///  `Kind::extend`.)
    pub fn extend_packed(self, next: Kind) -> (Kind, usize) {
        let new_size = self.size + next.size;
        (Kind { size: new_size, ..self }, self.size)
    }

    /// Creates a `Kind` describing the record that can hold a value
    /// of the same kind as `self`, but that also is aligned to
    /// alignment `align`.
    ///
    /// Behavior undefined if `align` is not a power-of-two.
    ///
    /// If `self` already meets the prescribed alignment, then returns
    /// `self`.
    ///
    /// Note that this method does not add any padding to the overall
    /// size, regardless of whether the returned kind has a different
    /// alignment. You should be able to get that effect by passing
    /// an appropriately aligned zero-sized type to `Kind::extend`.
    pub fn align_to(self, align: usize) -> Kind {
        if align > self.align {
            Kind { align: align, ..self }
        } else {
            self
        }
    }

    /// Returns the amount of padding we must insert after `self`
    /// to ensure that the following address will satisfy `align`.
    ///
    /// Behavior undefined if `align` is not a power-of-two.
    ///
    /// Note that for this to make sense, `align <= self.align`
    /// otherwise, the amount of inserted padding would need to depend
    /// on the particular starting address for the whole record.
    fn pad_to(self, align: usize) -> usize {
        debug_assert!(align <= self.align);
        let len = self.size;
        let len_rounded_up = (len + align - 1) & !(align - 1);
        return len_rounded_up - len;
    }

    /// Creates a `Kind` describing the record for `self` followed by
    /// `next`, including any necessary padding to ensure that `next`
    /// will be properly aligned. Note that the result `Kind` will
    /// satisfy the alignment properties of both `self` and `next`.
    ///
    /// Returns `(k, offset)`, where `k` is kind of the concatenated
    /// record and `offset` is the start of the `next` embedded witnin
    /// the concatenated record (assuming that the record itself
    /// starts at offset 0).
    pub fn extend(self, next: Kind) -> (Kind, usize) {
        let new_align = cmp::max(self.align, next.align);
        let realigned = Kind { align: new_align, ..self };
        let pad = realigned.pad_to(new_align);
        let offset = self.size + pad;
        let new_size = offset + next.size;
        (Kind { size: new_size, align: new_align, tracked: self.tracked || next.tracked }, offset)
    }

    /// Creates a `Kind` describing the record for `n` instances of
    /// `self`, with a suitable amount of padding between each.
    pub fn array(self, n: usize) -> Kind {
        let padded_size = self.size + self.pad_to(self.align);
        Kind { size: padded_size * n, ..self }
    }

    /// Creates a `Kind` describing the record for `n` instances of
    /// `self`, with no padding between each.
    pub fn array_packed(self, n: usize) -> Kind {
        Kind { size: self.size * n, ..self }
    }
}
```

Next we have the allocator trait itself.

Much of this is inspired by the previous two allocator RFCs, [RFC PR 39][] and [RFC PR 244][].

```rust
#[derive(Copy, Clone, Debug)]
pub struct AllocError;

/// An implementation of `Allocator` can allocate, reallocate, and
/// deallocate arbitrary blocks of data described via `Kind`.
/// However, if an implementor does not also implement `TrackedAllocator`,
/// then it is not guaranteed to handle inputs that `require_tracked`.
///
/// Some of the methods require that a kind *fit* a memory block.
/// What it means for a kind to "fit" a memory block means is that
/// the following two conditions must hold:
///
/// 1. The block's starting address must be aligned to `kind.align()`.
///
/// 2. The block's size must fall in the range `[orig, usable]`, where:
///
///    * `orig` is the size last used to allocate the block, and
///
///    * `usable` is the capacity that was (or would have been)
///      returned when (if) the block was allocated via a call to
///      `alloc_excess` or `realloc_excess`.
///
/// Note that due to the constraints in the methods below, a
/// lower-bound on `usable` can be safely approximated by a call to
/// `usable_size`.
pub trait Allocator {

    /// Returns a pointer suitable for holding data described by
    /// `kind`
    ///
    /// If `kind.requires_tracking()`, then may panic or signal an
    /// allocation failure if this allocator does not support tracking
    /// (if neither, then it must proceed as follows).
    ///
    /// Returns a pointer suitable for holding data described by
    /// `kind`, meeting its size and alignment guarantees.
    ///
    /// if (1.) allocation succeeds, (2.) `kind.requires_tracking()`
    /// and (3.) a garbage collector is linked in, then the returned
    /// storage will be initialized to zero before the address is
    /// returned.
    ///
    /// Returns null if allocation fails.
    ///
    /// Behavior undefined if `kind.size()` is 0 or if `kind.align()`
    /// is larger than the largest platform-supported page size.
    unsafe fn alloc(&mut self, kind: Kind) -> Address;

    /// Deallocates the memory referenced by `ptr`.
    ///
    /// `ptr` must have previously been provided via this allocator,
    /// and `kind` must *fit* the provided block.
    unsafe fn dealloc(&mut self, ptr: Address, kind: Kind);

    /// Returns the minimum guaranteed usable size of a successful
    /// allocation created with the specified `kind`.
    ///
    /// Clients who wish to make use of excess capacity are encouraged
    /// to use the `alloc_excess` and `realloc_excess` instead, as
    /// this method is constrained to conservatively report a value
    /// less than or equal to the minimum capacity for *all possible*
    /// calls to those methods.
    ///
    /// However, for clients that do not wish to track the capacity
    /// returned by `alloc_excess` locally, this method is likely to
    /// produce useful results.
    unsafe fn usable_size(&self, kind: Kind) -> Capacity { kind.size }

    /// Extends or shrinks the allocation referenced by `ptr` to
    /// `new_size` bytes of memory, retaining the alignment `align`
    /// and value-tracking state specified by `kind`.
    ///
    /// `ptr` must have previously been provided via this allocator.
    ///
    /// `kind` must *fit* the `ptr` (see above).
    ///
    /// If this returns non-null, then the memory block referenced by
    /// `ptr` may have been freed and should be considered unusable.
    ///
    /// Returns null if allocation fails; in this scenario, the
    /// original memory block referenced by `ptr` is unaltered.
    ///
    /// Behavior undefined if above constraints are unmet. Behavior
    /// also undefined if `new_size` is 0.
    unsafe fn realloc(&mut self, ptr: Address, kind: Kind, new_size: Size) -> Address {
        if new_size <= self.usable_size(kind) {
            return ptr;
        } else {
            let new_ptr = self.alloc(Kind { size: new_size, ..kind });
            if !new_ptr.is_null() {
                ptr::copy(ptr as *const u8, new_ptr, cmp::min(kind.size, new_size));
                self.dealloc(ptr, kind);
            }
            return new_ptr;
        }
    }

    /// Behaves like `fn alloc`, but also returns the whole size of
    /// the returned block. For some `kind` inputs, like arrays, this
    /// may include extra storage usable for additional data.
    unsafe fn alloc_excess(&mut self, kind: Kind) -> Excess {
        Excess(self.alloc(kind), self.usable_size(kind))
    }

    /// Behaves like `fn realloc`, but also returns the whole size of
    /// the returned block. For some `kind` inputs, like arrays, this
    /// may include extra storage usable for additional data.
    unsafe fn realloc_excess(&mut self, ptr: Address, kind: Kind, new_size: Size) -> Excess {
        Excess(self.realloc(ptr, kind, new_size),
               self.usable_size(Kind { size: new_size, ..kind }))
    }

    /// Allocates a block suitable for holding an instance of `T`.
    ///
    /// Captures a common usage pattern for allocators.
    unsafe fn alloc_one<T>(&mut self) -> Result<Unique<T>, AllocError> {
        let p = self.alloc(Kind::new::<T>()) as *mut T;
        if !p.is_null() { Ok(Unique::new(p)) } else { Err(AllocError) }
    }

    /// Deallocates a block suitable for holding an instance of `T`.
    ///
    /// Captures a common usage pattern for allocators.
    unsafe fn dealloc_one<T>(&mut self, ptr: Unique<T>) {
        self.dealloc(*ptr as *mut u8, Kind::new::<T>());
    }

    /// Allocates a block suitable for holding `n` instances of `T`.
    ///
    /// Captures a common usage pattern for allocators.
    unsafe fn alloc_array<T>(&mut self, n: usize) -> Result<Unique<T>, AllocError> {
        let p = self.alloc(Kind::new::<T>().array(n)) as *mut T;
        if !p.is_null() { Ok(Unique::new(p)) } else { Err(AllocError) }
    }

    /// Deallocates a block suitable for holding `n` instances of `T`.
    ///
    /// Captures a common usage pattern for allocators.
    unsafe fn dealloc_array<T>(&mut self, ptr: Unique<T>, n: usize) {
        self.dealloc(ptr as *mut u8, Kind::new::<T>().array(n));
    }

    /// Allocator-specific method for signalling an out-of-memory
    /// condition.
    ///
    /// Any activity done by the `oom` method must not allocate
    /// from `self` (otherwise you essentially infinite regress).
    unsafe fn oom(&mut self) -> ! { ::std::intrinsics::abort() }
}
```

## Stabilization Path

We do not want to prematurely stabilize API's related to garbage
collection. Therefore, this RFC suggests that we do not stabilize the
`Tracked` bound nor intrinsics related to it until we have more
concrete experience with collector integration.  However, we *do* want
to provide allocator parameterization as soon as we can, potentially
sooner than we have sufficient experience with collector integration
for complete stabilization of all of the above APIs.  Therefore, this
RFC suggests a stabilization path where we plan to mark the
`Allocator` trait as stable, as well as the
[`Untracked` marker trait][classification].

## Tracking and Trait Objects
[tracking and trait objects]: #tracking-and-trait-objects

This RFC is designed to statically categorize all types according to
whether their values need to be tracked.

At first it may seem trivial: just recursively traverse the structure
of a type, descending into its fields, and see if any type found
implements `Tracked`.

 * The recursive descent referenced above includes the `T` in
   `PhantomData<T>` and `*const T`/`*mut T`, but not does not include
   the types of `&`- nor `&mut`-references, which are never
   interpreted as owning their elemeents.

However, one place where static categorization falls short is trait
objects.

For example, we might have the following:

```rust
fn make_object_with_roots() -> Box<Trait> {
  let has_roots: Box<HasRoots> = Box::new(HasRoots { ... });
  has_roots as Box<Trait>
}
```

In the above code, we are returning a boxed value that contains roots,
but there is nothing in the type `Box<Trait>` to indicate that.

Therefore, we must assume an object type `Box<Trait>` may hold tracked
values (and therefore also `Vec<Box<Trait>>`, or
`struct Foo(u32, Box<Trait>)`, et cetera).

To circumvent this in cases where a user of trait-objects wants to
absolutely ensure no tracked values leak in, `Untracked` will be
available as a built-in object bound. One can write
`Box<Trait+Untracked>` for the type of such an object.

## "Waiter, I Didn't Order That"
[didnt order that]: #waiter-i-didnt-order-that

One of the main goals of this design was to ensure that developers and
programs that do not use garbage collection should not pay for it.

There are two design axes on the RFC that are intended to address
this: One is achieving nearly zero-cost runtime overhead,
and the other is managing the programming complexity.

### Runtime Overhead

From the runtime overhead perspective: Even though the low-level
memory manager (aka `#[allocator]`) needs to keep meta-data for
mapping interior addresses to their values, this RFC is carefully
set up so that the `#[allocator]` need *only* do this when it is
explicitly asked to do so via the allocation methods with
[the `_tracked` suffix][low-level allocator api].

The high-level allocators are expected to continue to invoke
the other *non-tracking* allocation for any type that
is untracked.

I hope, as the RFC author, that this is sufficient to ensure that
the value-tracking functionality does not impose any runtime
overhead on programs that do not use the GC and are written without
any awareness of the potential for GC.

The main potential exception to this principle are boxed trait
objects. One can write `Box<Trait+Untracked>` to work around this, as
noted in the [tracking and trait objects][] section, but this is admittedly a bit of
a wart in the design. (One that has been known for a long time.)

### Program Complexity

The hope here is that developers of container data structures will
simply just use the `Allocator` trait on their containers in the same
natural manner that they would have used the low-level `allocate` and
`deallocate` methods. They will not have to think about GC roots or
value-tracking (that will instead be the job of the implementor of the
allocator when/if they add the `TrackingAllocator` marker to their
allocator).

There *is* a certain cost to the hypothetical `Allocator` implementor.
Any `Allocator` implementation will need to ensure that when
it is given a request to allocate a `Kind` that requires tracking,
it either:

 1. sets up appropriate metadata to support the block descriptor
    and its value-reflection functionality via `BDFunc`,

 2. delegates all such tracked allocation requests to a different
    allocator that *does* support tracking (such as the host system
    allocator), or

 3. signals allocation failure when asked allocate tracked values.

# Drawbacks
[drawbacks]: #drawbacks

## Complexity

Obviously this system is more complex than the previous two
allocator RFCs, [RFC PR 39][] and [RFC PR 244][].

 * [RFC PR 39][] made no attempt to account for garbage collection; this
   was one reason why it was declined

 * [RFC PR 244][] attempted to account for garbage collection but did
   so in a naive manner; it assumed that memory with GC pointers could
   be tracked (e.g. in a linked list) and directly scanned for roots,
   without attempting to start from the program stack. That design
   fails to handle garbage cycles that cross from the GC heap through
   the explicit heap and back to the GC heap again (an example
   discussed in the [similar to tracing] section).

As Einstein said, "as simple as possible, but not simpler": If the
goal is to integrate with a future GC, the prior two RFC's fell into
the bucket of "simpler than possible"; this RFC may be more complex
than necessary, but it is not clear where to improve it without
introducing strong assumptions about user code and GC's.

## Overhead

Value-tracking adds overhead that an idealized system might be able to
avoid. This RFC has attempted to manage that overhead via static
type-based categorization, but this may be insufficient.

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

## Variations on the above RFC

This subsection presents alternatives that are each individual,
potentially orthogonal variations on the RFC as written. They
represent relatively small changes to this design as given, rather
than "big picture" revisions of the whole approach to the problem.

## `#[tracked_allocator]` crate and `#[needs_tracked_allocator]` attribute

Instead of adding new methods to the `#[allocator]` crate API, it may be
significantly cleaner to instead set up a parallel attribute system
to the `#[needs_allocator]` and `#[allocator]` attributes from
[RFC 1183].

I have not really thought too carefully about this alternative; it only
occurred to me as I was writing up the RFC.

There are two reasons to consider doing this:

1. At first it may seem that it could be used to decouple value-tracking
   from the standard library entirely: Just put tracked allocators into
   a separate crate that is not even part of the standard library.
   That would certainly ensure that if a program does not use value-tracking,
   then it could not be paying for it within its low-level allocation routines
   (since the corresponding code would not even be linked in)!

   Unfortunately, since we cannot 100% categorize all values (see
   [tracking and trait objects][]), it seems unlikely that we could
   avoid putting `#[tracked_allocator]` into the standard library.

2. If there were a separate `#[tracked_allocator]` that were
   potentially built as a separate crate *on top* of the
   `#[allocator]` crate, then that could ease both the development of
   `#[allocator]` crates and of tracking allocators (which would grab
   their own backing storage from the underlyng `#[allocator]` crate).

You may be able to infer that I see the second point above as a strong
motivation to investigate this alternative, if necessary as an
amendment incorporated during the implementation of the `#[allocator]`
parts of the RFC.

## Add a `is_tracked_val` intrinsic

The [classification][] section proposes two main ways to classify
whether a given input cannot possibly carry a tracked value: an
`Untracked` OIBIT, and an `is_tracked::<T>()` intrinsic.  The latter
works solely on types.

A problem with relying on these two API's alone is that they
need to be conservative, which menas they need to always categorize
trait objects like `Box<Trait>` as holding tracked values.

One way that we might try to work around this would be to take
inspiration from how `sizeof` is handled for trait objects: We could
add an intrinsic, `is_tracked_val::<T: ?Sized>(val: &T) -> bool`, that
returns false only if `*val` can *never* have tracked values embedded
within itself or any values it owns.

So for example, then we might have the following:
```rust
let has_roots: Box<HasRoots> = Box::new(HasRoots { ... });
let may_roots: Box<Option<HasRoots>> = Box::new(None);
let no_roots:  Box<i32> = Box::new(42);

is_tracked_val(has_roots as Box<Trait>); // ==> true
is_tracked_val(may_roots as Box<Trait>); // ==> true
is_tracked_val(no_roots  as Box<Trait>); // ==> false
```

(Note that `may_roots` returns `true` above: even though the box does
not *currently* have any tracked values within it, it might in the
future, and so the intrinsic indicates that we should be ready to scan
it in the future.)

----

The above seems like a worthwhile idea, but it is important to
understand how far it should go.

In particular, what should happen here:

```rust
let no_roots: Box<i32> = Box::new(42);
let a = no_roots as Box<Trait>;
let recur: Box<(i32, Box<Trait>)> = Box::new((42, a));
is_tracked_val(recur as Box<Trait>); // ==> ???
```

What should the `is_tracked_val` intrinsic produce here?

 * If the goal is to try to use the `is_tracked_val` intrinsic to
   recover the lost part of the zero-cost principle (i.e., that using
   trait objects in code that never uses `Tracked` types should not
   have to pay for value tracking), then that would imply that the
   intrinsic should recursively descend into `recur` and query the box
   it holds to ask if it might ever hold any tracked values.

   (Of course, recursive descent may cause problems elsewhere, but
    let us ignore that for now.)

 * But remember the earlier example of `may_roots` -- the intrinsic is
   not supposed to be concerned solely with the current contents of
   the value, but also with any potential *future* contents.  It does
   not matter that `recur` currently holds `no_roots` in the
   `Box<Trait>` part of the tuple; at some point in the future,
   someone might swap in a `Box<HasRoots> as Box<Trait>` in there.

Therefore, even if we *were* to add the `is_tracked_val` intrinsic
(which may indeed be a good idea), we still might have unexpected
sources of value-tracking.

## Separate methods on tracking allocator

The RFC as written puts all of the allocation trait methods into the
single `Allocator` trait, and then adds the `TrackedAllocator`
extension as a way to narrow the specification of each of those
methods (specifically, the `TrackedAllocator` marker trait removes the
option for an allocator to cause compilation-failure or runtime-panic
when given a `Tracked` type as input).

Another option would be to make the traits be `UntrackedAllocator` and
`TrackedAllocator`. `UntrackedAllocator` would provide one set of
methods that solely work on types that implement the `Untracked`
trait, and `TrackedAllocator` would provide a set of methods that work
with all types.

 * (Note that in this scenario we would still keep the property that
   `trait TrackedAllocator: UntrackedAllocator { ... }` -- it would
   just be more explicit at each method invocation whether one was
   allocating a tracked or untracked block of memory.)

The problem with this latter design is that it raises the question:
What should the standard library `Vec<T, A>` use as the bound for the
`Allocator` parameter `A`?

 * In principle we want `Vec` to work with all kinds of values, so
   that we can compose together things like `Vec<HasRoots>`. This
   implies that `Vec` needs to take a `TrackedAllocator` in
   general.

 * But we also want `Vec` to be able to work with client-programmed
   allocators that do not support value tracking. This implies that
   `Vec` needs to take an `UntrackedAllocator` in general.

----

It is *possible* that the [specialization RFC][RFC PR 1210] would
allow us to write separate impls for both of the above cases, e.g.:

```rust
impl<U, A> Vec<U, A> where U: Untracked, A: UntrackedAllocator { ... }
impl<T, A> Vec<T, A> where T, A: TrackedAllocator { ... }
```

but I am not certain of this, especially since `TrackedAllocator` is a
subtrait of `UntrackedAllocator`, and I am not sure whether
specialization would allow the combination of `impl`s above (nor am I
sure which `impl` it would resolve to when combining an `Untracked`
value with a `TrackedAllocator`).

----

In any case, it certainly seems *simpler* to just provide a single
`Allocator` interface that all containers are expected to program to,
even if it may reduce the amount of potential static reasoning encoded
via the type system.

## Mark bits for the root identification scan

The protocol for scanning values in order to identify all of the roots
in the owned object graph does not provide a built-in way to mark values
as "already scanned" during the run.

If the owned object graph is in fact a tree, then this works fine: we
incrementally uncover new objects as we walk the tree, but we will
never encounter the same object more than once.

If the owned object graph is a directed acyclic graph (DAG), then that
native scan is not so good: you may waste effort repeatedly scanning
the state owned by the same value repeatedly. (This leads to
exponential blow up in a pathological case.)

 * A case where one sees a DAG occur is something like when there are
   many `Rc<Vec<HasRoots>>` on the stack that all point to the same
   large vector. (The exponential blowup occurs with a chain of such
   shared references, where each link on the chain has multiple owners
   reachable from the prior owners.)

And of course, if the object graph has cycles in the ownership, then
the naive scan will infinite loop.

One way to avoid the problems that occur with DAG's and cycles would
be to require every tracked value to have an additional mark bit
stored somewhere with its block (not necessarily with the value
itself). Then, each run of the root scanning process would first clear
all the bits for *every* allocated block, and then use the bits as it
walks through the values to ensure that it visits each value at most
once during a root scan.

 * (This would make root identification even more like tracing than
    [it already is][similar to tracing])

But, it is not strictly necessary for the value-walking protocol to
provide mark-bits. The GC system can itself keep its own tables of
what values it has seen previously during the root scanning process.

 * It seems possible that the cost in time of clearing all the bits
   for every allocated block could add overhead comparable to that of
   maintaining such a table of visited values.

 * Embedded mark bits would avoid the cost in space of building up
   that table at the start of GC, and instead distributing the space
   cost across all tracked values. If this were a language where all
   memory were GC'ed (and thus we would want to ensure that the GC can
   run under very tight memory conditions), then this would be a very
   serious concern; but I do not think is as strong an issue for Rust,
   where I anticipate the majority of memory will not be GC-managed.

This RFC is currently opting for a simpler allocation protocol, rather
than baking in a marking scheme that may not even be the right choice
in the long run anyway.

# Unresolved questions
[unresolved questions]: #unresolved-questions

* How will GC actually work? As noted in the [GC details out of scope][]
  section, this question is not actually meant to be fully answered
  here. Still, I welcome people to point out flaws in the system
  described above, especially with respect to the hypothesized GC
  integrations.

* The `Kind` API added here is something new that was not part of
  [RFC PR 39][] and [RFC PR 244][]. Personal experience provides some
  evidence that it can simplify allocation code (it captures
  common patterns for size and alignment calculations), but does it
  suffice?

# Appendices
[appendices]: #appendices

## Bibliography
[Bibliography]: #bibliography

* Daniel Micay, 2014. RFC: Allocator trait. https://github.com/rust-lang/rfcs/blob/ad4cdc2662cc3d29c3ee40ae5abbef599c336c66/active/0000-allocator-trait.md
[RFC PR 39]: https://github.com/rust-lang/rfcs/blob/ad4cdc2662cc3d29c3ee40ae5abbef599c336c66/active/0000-allocator-trait.md

* Felix Klock, 2014, RFC: Allocator RFC, take II. https://github.com/pnkfelix/rfcs/blob/fsk-allocator-rfc/active/0000-allocator.md
[RFC PR 244]: https://github.com/rust-lang/rfcs/blob/d3c6068e823f495ee241caa05d4782b16e5ef5d8/active/0000-allocator.md

* Alex Crichton, 2015, RFC: Allow changing the default allocator. https://github.com/rust-lang/rfcs/blob/master/text/1183-swap-out-jemalloc.md
[RFC 1183]: https://github.com/rust-lang/rfcs/blob/master/text/1183-swap-out-jemalloc.md

* Aaron Turon, 2015, RFC: impl specialization. https://github.com/aturon/rfcs/blob/impl-specialization/text/0000-impl-specialization.md
[RFC PR 1210]: https://github.com/aturon/rfcs/blob/impl-specialization/text/0000-impl-specialization.md

## Model for GC integration
[Model for GC integration]: #model-for-gc-integration

Convention: `Managed` represents a value that is meant to solely be
allocated to the GC-managed heap. `HasRoots` represents a value that
might hold a reference to garbage-collected data.  `Interior` stands
for some value that is embedded as a field within another struct type.

This RFC assumes the following model for GC integration:

The system memory can be divided into four parts:
 * The program register and thread stacks
 * Value-tracked heap
 * Untracked heap
 * GC-managed heap

Each value allocated in value-tracked heap (or just "tracked") has
some overhead associated with it. In particular, the runtime maintains
a data structure that maps tracked memory addresses to the metadata
associated with that block of memory. This way, if the current program
state holds a `*const Interior` into the interior of a block, the GC
can still map from the `*const Interior` to the entire block. (This is
necessary because that `*const Interior` pointer may be used by the
program to reconstruct the pointer to the whole structure, so it is
necessary to treat the whole structure as live.)

We assume there is some overhead associated with maintaining the
data-structure for the value-tracked map. To ensure the property that
programs do not pay for unused features, we add the untracked heap.
Values allocated to the untracked heap have no exposed structure for
metadata lookup.

The choice of whether a value is allocated to the tracked or untracked
heap is driven by separate functions in the `#[allocator]` crate API.
Thus, clients using the `#[allocator]` crate must properly dispatch to
the appropriate method based on what types they are handling (and also
potentially based on whether GC support has been globally disabled;
see [section "I didn't order that"][didnt order that]).

```
   Untracked            Stack(s)            Tracked            GC-Managed       Supported?
----------------    ----------------    ----------------    ----------------    ----------
                     Vec<HasRoots> ----> [HasRoots; n] -----> ManagedValue         YES

                     Box<HasRoots> -----> HasRoots ---------> ManagedValue         YES

                    *const Interior ---> [HasRoots; n] -----> ManagedValue         YES

                                              Boxed<T>  <---- ManagedValue         YES

                                              HasRoots  <---- ManagedValue <-+ (garbage)
                                              |                              |
                                              +-------------> ManagedValue --+     YES!!

                       HasRoots   --------------------------> ManagedValue        MAYBE

                    Vec<_, GcAlloc> ----------------------> [ManagedVals; n]      MAYBE

      HasRoots  <---  *HasRoots
       |
       +----------------------------------------------------> ManagedValue         NO

```

The `YES!!` next to the case where GC-managed value is pointing to the
tracked `HasRoots` that then refers back to the is indicating a case
where garbage can be collected even when the cycle goes through
explicitly managed memory. This is the case discussed in the
[similar to tracing] section.

The `MAYBE` next to the lines that directly connects a stack-allocated
value to a GC-managed value is due to ... complications.

Essentially, the "MAYBE" means "we might be able to support it, but
only if the garbage collector is itself more deeply integrated with
the Rust runtime and allocator. This RFC does not describe all the
details of what would be necessary for such integration; this is
further discussed in the [GC details out of scope][] section.
