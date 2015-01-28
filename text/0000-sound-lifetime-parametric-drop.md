- Start Date: 2013-08-29
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Remove `#[unsafe_destructor]` from the Rust language.  Make it safe
for developers to implement `Drop` on type- and lifetime-parametric
structs and enum by imposing new rules on code where such types occur,
to ensure that the drop implementation cannot possibly read or write
data via a reference of type `&'a Data` where `'a` could have possibly
expired before the drop code runs.

Note: This RFC is describing a feature that has been long in the
making; in particular it was previously sketched in Rust [Issue #8861]
"New Destructor Semantics" (the source of the tongue-in-cheek "Start
Date" given above), and has a [prototype implementation] that is being
prepared to land.  The purpose of this RFC is two-fold:

 1. standalone documentation of the (admittedly conservative) rules
    imposed by the new destructor semantics, and

 2. elicit community feedback on the rules, both in the form they will
    take for 1.0 (which is relatively constrained) and the form they
    might take in the future (which allows for hypothetical language
    extensions).

[Issue #8861]: https://github.com/rust-lang/rust/issues/8861

[prototype implementation]: https://github.com/pnkfelix/rust/tree/77afdb70a1d4d5a20069f12412bfeda3ccd145bf

# Motivation

Part of Rust's design is rich use of Resource Acquisition Is
Initialization (RAII) patterns, which requires destructors: code
attached to certain types that runs only when a value of the type goes
out of scope or is otherwise deallocated. In Rust, the `Drop` trait is
used for this purpose.

Currently (Rust 1.0alpha), a developer cannot implement `Drop` on a
type- or lifetime-parametric type (e.g. `struct Sneetch<'a>` or `enum
Zax<T>`) without attaching the `#[unsafe_destructor]` attribute to
it. The reason this attribute is required is that the current
implementation allows for such destructors to inject unsoundness
accidentally (e.g. reads from or writes to deallocating memory).

Furthermore, while some destructors can be implemented with no danger
of unsoundness, regardless of `T` (assuming that any `Drop`
implementation attached to `T` is itself sound), as soon as one wants
to interact with borrowed data within the (e.g. access a field `&'a
StarOffMachine` from a value of type `Sneetch<'a>` ), there is
currently no way to enforce a rule that `'a` *strictly outlive*` the
value itself. This is a huge gap in the language as it stands: as soon
as a developer attaches `#[unsafe_destructor]` to such a type, it is
imposing a subtle and *unchecked* restriction on clients of that type
that they will not ever allow the borrowed data to expire first.

Concretely: If today Sylvester writes:

```rust
// opt-in to the unsoundness!
#![feature(unsafe_destructor)]

pub mod mcbean {
    use std::cell::Cell;

    pub struct StarOffMachine {
        usable: bool,
        dollars: Cell<u64>,
    }
    
    impl Drop for StarOffMachine {
        fn drop(&mut self) {
            let contents = self.dollars.get();
            println!("Dropping a machine; sending {} dollars to Sylvester.",
                     contents);
            self.dollars.set(0);
            self.usable = false;
        }
    }

    impl StarOffMachine {
        pub fn new() -> StarOffMachine {
            StarOffMachine { usable: true, dollars: Cell::new(0) }
        }
        pub fn remove_star(&self, s: &mut Sneetch) {
            assert!(self.usable,
                    "No different than a read of a dangling pointer.");
            self.dollars.set(self.dollars.get() + 10);
            s.has_star = false;
        }
    }

    pub struct Sneetch<'a> {
        name: &'static str,
        has_star: bool,
        machine: Cell<Option<&'a StarOffMachine>>,
    }

    impl<'a> Sneetch<'a> {
        pub fn new(name: &'static str) -> Sneetch<'a> {
            Sneetch {
                name: name,
                has_star: true,
                machine: Cell::new(None)
            }
        }

        pub fn find_machine(&self, m: &'a StarOffMachine) {
            self.machine.set(Some(m));
        }
    }

    #[unsafe_destructor]
    impl<'a> Drop for Sneetch<'a> {
        fn drop(&mut self) {
            if let Some(m) = self.machine.get() {
                println!("{} says ``before I die, I want to join my \
                          plain-bellied cousins.''", self.name);
                m.remove_star(self);
            }
        }
    }
}

fn unwary_client() {
    use mcbean::{Sneetch, StarOffMachine};
    let s1 = Sneetch::new("Sneetch One");
    let m = StarOffMachine::new();
    let s2 = Sneetch::new("Sneetch Two");
    let s3 = Sneetch::new("Sneetch Zee");

    s1.find_machine(&m);
    s2.find_machine(&m);
    s3.find_machine(&m);
}

fn main() {
    unwary_client();
}
```

In Sylvester's code, the `Drop` implementation for `Sneetch` invokes a
method on the borrowed reference in the field `machine`. This implies
there is an implicit restriction on an value `s` of type
`Sneetch<'a>`: the lifetime `'a` must *strictly outlive* `s`.

(The example encodes this constraint in a dynamically-checked manner
via an explicit `usable` boolean flag that is (only) set to false in
the machine's own destructor; it is important to keep in mind that
this is just a way to illustrate the violation: Using a machine after
`usable` is set to false is analogous to dereferencing a `*mut T` that
has been deallocated, or similar soundness violations.)

Sylvester's API does not encode the constraint "`'a` must strictly
outlive the `Sneetch<'a>`" explicitly; Rust currently has no way of
expressing the constraint that one lifetime be strictly greater than
another lifetime or type (the form `'a:'b` only formally says that
`'a` must live *at least* as long as `'b`).

Thus, client code like that in `unwary_client` can inadvertantly set
up scenarios where Sylvester's code may break, and Sylvester might be
completely unaware of the vulnerability.

(The `Sneetch<'a>` example may seem contrived, and it is; see
[Appendix A] for a far less contrived example, that also illustrates
how the use of borrowed data can lie hidden behind type parameters.)

----

This RFC is proposes to fix this scenario, by adding the notion of
"strictly outlives" to the compiler internals (though not to the
language itself, which is unnecessary at this time), and imposing the
following consevative rule:

Let `e` be some expression; if the type of `e` has type `D` or owns data of type `D`, where `D`

  * (1.) has a parametric `Drop` impl, and

  * either:

     * (2a.) that `Drop` impl is instantiated at `'a` (directly), or

     * (2b.) that `Drop` impl has a type with a bound `T` where `T`
             is some trait that has at least one method, then,

then `'a` must strictly outlive the scope of `e`.


This rule catches the known cases where a destructor could possibly
reference borrowed data via a reference of type `&'a _` or `&'a mut_`.

At the same time, this rule allows much existing code which *is*
sound, to compile without complaint from rustc.

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar
with the language to understand, and for somebody familiar with the compiler to implement.
This should get into specifics and corner-cases, and include examples of how the feature is used.

# Drawbacks

Why should we *not* do this?

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

What parts of the design are still TBD?

# Appendices

## Appendix A: Why and when would Drop read from borrowed data
[Appendix A]: #appendix-a-why-and-when-would-drop-read-from-borrowed-data

TODO / TOFINISH

Here is a story: Julia inherited some code, and it is misbehaving.  It
appears like key/value entries that she inserts into the standard
library's `HashMap` are not always retrievable from the map, and
Julia's current hypothesis is that something is causing the keys'
computed hash codes to change dynamically, sometime after they have
been inserted into the map (but it is not obvious when). Julia thinks
this hypothesis is plausible, but does not want to audit all of the
key types for possible causes of hash code corruption until after she
has hard evidence confirming the hypothesis.

It is not hard to write code that walks the hash map and checks that
all of the keys produce a hash code that is consistent with their
location in the map.  However, since it is not clear when the keys'
hash codes are changing, it is not clear where in the code such checks
should be added. (The hash map is sufficiently large that she cannot
simply add calls to do this consistency check everywhere.)

However, there is one spot in the control flow that is a clear
contender: if the check is run right before the hash map is dropped,
then that would surely be sometime after the hypothesized corruption
had occurred.  In other words, Julia could make her own local copy of
the hash map library and add this check to a `impl<K,V,S> Drop for
HashMap<K,V,S> { ... }` implementation.

Using this, Julia manages confirms her hypothesis, and puts this
variation of HashMap up on `crates.io`, calling the new type the
`CheckedHashMap`.
