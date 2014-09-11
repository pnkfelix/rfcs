- Start Date: 2014-09-11
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Make all arms in a `match` process the input in the same way.

 * If any one arm binds some substructure by-move, then the whole
   match must take the input by-move.

 * If any one arm binds some substructure by-reference, then the whole
   match must take the input by-reference.

Bindings that make copies of their matched substructure are not
affected by this change; that is, (1) they continue to make copies of
their part of the input, and (2) they are compatible with both ways of
taking the input (by-reference or by-move). Thus copying bindings
introduce no constraints on how the overall match takes its input.
Likewise, `match` expressions that have no bindings in any of their
arms are not affected by this change.

# Motivation

The main reason for making this change is that it simplifies program
analysis for a human reader.  Right now, when one sees an expression
`match input { ... }`, one must reason about each arm of the match
individually to know whether the value of `input` is moved into a new
location, or if it is left in its original location (and potentially
modified in place).  The other arms have independent behavior.

This change would simplify that reasoning: if any arm in a match moved
the input, then all must.

This change was originally proposed in an early draft of
[RFC PR 210: Static drop semantics][RFC PR 210];
it was there referred to as the "match-arm rule".  The "match-arm
rule" was later removed from the that RFC (it was determined to be not
strictly necessary for that RFC), but I think this change is still
worth considering on its own for 1.0; we can relax the restriction
proposed by this RFC after 1.0 if we determine it necessary.

# Detailed design
[Detailed design]: #detailed-design

## Review / Background

Today (without this RFC), each arm in a match expression can be categorized
as follows: `NonBinding`, `Copying`, `Borrowing`, and `Moving`.
The categorization can derived from the patterns in the arm, by
using the following lattice structure:

```
                 Conflicting
                  /      \
                 /       \
           Borrowing    Moving
                \        /
                \       /
                 Copying
                    |
                NonBinding
                    |
                 Unknown
```

Initially, patterns have `Unknown` state.  One assigns lattice values
to primitive patterns by following these rules: A `_` pattern is
`NonBinding`.  A `ref <identifier>` pattern is `Borrowing`.  A non-ref
identifier pattern for substructure that implements the `Copy` bound
(e.g. `int`) is `Copying`; a non-ref identifier pattern for other
substructure is `Moving`.

Then, one follows the lattice structure to determine the overall
category for the pattern.  If there is both `Borrowing` and `Moving`
subpatterns in the same pattern, then the overall pattern is `Conflicting`,
which marks it as illegal.

(Note that all of the above is just a review; it reflects the state of
Rust today.)

Some examples:

 * `(_, some_int)` pattern is Copying, since
   NonBinding + Copying => Copying

 * `(some_int, some_box)` pattern is Moving, since
   Copying + Moving => Moving

 * `(ref x, some_box)` pattern is Conflicting, since
   Borrowing + Moving => Conflicting

## Proposed change

The main change that this RFC is proposing is this: instead of just
doing this analysis for each match-arm separately, do it for the whole
match expression, combining the lattice values of distinct arms to
determine the categorization for the whole `match`.

The compiler rejects any `match` expression that is categorized as
`Conflicting`.

# Drawbacks

Some code that is legal today might stop working.  pnkfelix encountered
very few such cases when bootstrapping his work on [RFC PR 210]; see [Impact on real code].


# Alternatives

## Require a syntactic marker at the input site

The main motivation for this change was simplifying program analysis
for a human reader.  However, even with this change, one must still
scan the match arms to determine whether the match will take its input
by-reference or by-move.

We could go further, and actually syntactically require the programmer
to indicate whether a match by-reference or by-move was intended.
Here is a hypothetical syntax for this:
```
// You write this:
match ref <input> { // `ref`: input always stays in place
   ... // we can use either `ref` or copying bindings within here
}

// Or you write this:
match <input> { // absence of `ref`: input is moved or copied (depending on `Copy`)
   ... // you cannot use any `ref` bindings within here
}
```

So with this, the programmer would know from just looking at the first
token after `match` whether they are looking at something that
consumes the input or not.

But, there is a drawback: This alternative proposal must address the
question of how non-binding matches should be treated.  Intutively, if
the goal is to make it apparent at start of the `match` expression
whether the input could be moved, then it seems like even non-binding
matches should require the `ref` keyword.  But that also sounds like a
serious step backwards with respect to usability. (And if you *don't*
require the `ref` keyword on non-binding matches, then you lose the
main property that this proposal was trying to achieve: you cannot
tell without looking at the arms themselves whether the input is
consumed by the `match` or not.)

## Make it a lint

This could arguably just be a style-guideline, not something
fundamental to the language.

## Status quo

We could just keep things the same.  It may complicate things for
adopting [RFC PR 210: Static drop semantics]

# Unresolved questions

## Impact on real code
[Impact on real code]: #impact-on-real-code

pnkfelix put in an analysis analogous to the one in the [Detailed
design].  But it is possible there is more code in the wild that would
be affected by this change.  

[RFC PR 210]: https://github.com/pnkfelix/rfcs/blob/fsk-nzdrop-rfc/active/0000-remove-drop-flag-and-zeroing.md