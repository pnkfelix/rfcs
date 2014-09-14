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
                 /        \
           Borrowing    Moving
                \        /
                 \      /
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

Some code that is legal today will stop working.
The notes from pnkfelix on [RFC PR 210] were based on a control-flow
sensitive variant of tahis change, which underestimates the effects
of the control-flow insensitive variant proposed here.
see [Impact on real code].


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

After hacking a control-flow insensitive version of the "match-arm
rule" into rustc, pnkfelix found the following cases flagged by the
`rustc` bootstrapping itself:


* [std::io::fs 482]
```
src/libstd/io/fs.rs:482:19: 486:10 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/libstd/io/fs.rs:482         let amt = match reader.read(buf) {
src/libstd/io/fs.rs:483             Ok(n) => n,
src/libstd/io/fs.rs:484             Err(ref e) if e.kind == io::EndOfFile => { break }
src/libstd/io/fs.rs:485             Err(e) => return update_err(Err(e), from, to)
src/libstd/io/fs.rs:486         };
```
[std::io::fs 482]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/libstd/io/fs.rs#L482-L486

* [std::io::fs 778]
```
src/libstd/io/fs.rs:778:17: 782:18 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/libstd/io/fs.rs:778                 match update_err(unlink(&child), path) {
src/libstd/io/fs.rs:779                     Ok(()) => (),
src/libstd/io/fs.rs:780                     Err(ref e) if e.kind == io::FileNotFound => (),
src/libstd/io/fs.rs:781                     Err(e) => return Err(e)
src/libstd/io/fs.rs:782                 }
```

[std::io::fs 778]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/libstd/io/fs.rs#L778-L782

* [std::io::fs 789]
```
src/libstd/io/fs.rs:789:13: 793:14 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/libstd/io/fs.rs:789             match result {
src/libstd/io/fs.rs:790                 Ok(()) => (),
src/libstd/io/fs.rs:791                 Err(ref e) if e.kind == io::FileNotFound => (),
src/libstd/io/fs.rs:792                 Err(e) => return Err(e)
src/libstd/io/fs.rs:793             }
```

[std::io::fs 789]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/libstd/io/fs.rs#L789-L793

* [std::io::util 177] -- (note that even control-flow sensitivity won't save this one)
```
src/libstd/io/util.rs:177:21: 181:22 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/libstd/io/util.rs:177                     match r.read(buf) {
src/libstd/io/util.rs:178                         Ok(len) => return Ok(len),
src/libstd/io/util.rs:179                         Err(ref e) if e.kind == io::EndOfFile => None,
src/libstd/io/util.rs:180                         Err(e) => Some(e),
src/libstd/io/util.rs:181                     }
```

[std::io::util 177]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/libstd/io/util.rs#L177-L181

* [std::io::util 228]
```
src/libstd/io/util.rs:228:19: 232:10 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/libstd/io/util.rs:228         let len = match r.read(buf) {
src/libstd/io/util.rs:229             Ok(len) => len,
src/libstd/io/util.rs:230             Err(ref e) if e.kind == io::EndOfFile => return Ok(()),
src/libstd/io/util.rs:231             Err(e) => return Err(e),
src/libstd/io/util.rs:232         };
```

[std::io::util 228]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/libstd/io/util.rs#L228-L232

* [std::io 692]
```
src/libstd/io/mod.rs:692:13: 696:14 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/libstd/io/mod.rs:692             match self.push_at_least(1, DEFAULT_BUF_SIZE, &mut buf) {
src/libstd/io/mod.rs:693                 Ok(_) => {}
src/libstd/io/mod.rs:694                 Err(ref e) if e.kind == EndOfFile => break,
src/libstd/io/mod.rs:695                 Err(e) => return Err(e)
src/libstd/io/mod.rs:696             }
```

[std::io 692]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/libstd/io/util.rs#L692-L696

* [std::io 1485]
```
src/libstd/io/mod.rs:1485:33: 1492:18 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/libstd/io/mod.rs:1485                 let available = match self.fill_buf() {
src/libstd/io/mod.rs:1486                     Ok(n) => n,
src/libstd/io/mod.rs:1487                     Err(ref e) if res.len() > 0 && e.kind == EndOfFile => {
src/libstd/io/mod.rs:1488                         used = 0;
src/libstd/io/mod.rs:1489                         break
src/libstd/io/mod.rs:1490                     }
                                                             ...
```

[std::io 1485]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/libstd/io/mod.rs#L1485-L1492


* [syntax::ext::env 64]
```
src/libsyntax/ext/env.rs:64:17: 71:6 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/libsyntax/ext/env.rs:64     let exprs = match get_exprs_from_tts(cx, sp, tts) {
src/libsyntax/ext/env.rs:65         Some(ref exprs) if exprs.len() == 0 => {
src/libsyntax/ext/env.rs:66             cx.span_err(sp, "env! takes 1 or 2 arguments");
src/libsyntax/ext/env.rs:67             return DummyResult::expr(sp);
src/libsyntax/ext/env.rs:68         }
src/libsyntax/ext/env.rs:69         None => return DummyResult::expr(sp),
                            ...
```

[syntax::ext::env 64]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/libsyntax/ext/env.rs#L64-L71

* [syntax::ext::tt::macro_rules 169]
```
src/libsyntax/ext/tt/macro_rules.rs:169:13: 204:14 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/libsyntax/ext/tt/macro_rules.rs:169             match parse(cx.parse_sess(), cx.cfg(), arg_rdr, mtcs.as_slice()) {
src/libsyntax/ext/tt/macro_rules.rs:170               Success(named_matches) => {
src/libsyntax/ext/tt/macro_rules.rs:171                 let rhs = match *rhses[i] {
src/libsyntax/ext/tt/macro_rules.rs:172                     // okay, what's your transcriber?
src/libsyntax/ext/tt/macro_rules.rs:173                     MatchedNonterminal(NtTT(tt)) => {
src/libsyntax/ext/tt/macro_rules.rs:174                         match *tt {
                                        ...
```

[syntax::ext::tt::macro_rules 169]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/libsyntax/ext/tt/macro_rules.rs#L169-L204


* [rustc::middle::resolve 3095]
```
rustc: x86_64-apple-darwin/stage1/lib/rustlib/x86_64-apple-darwin/lib/librustc
src/librustc/middle/resolve.rs:3095:9: 3158:10 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/librustc/middle/resolve.rs:3095         match module_prefix_result {
src/librustc/middle/resolve.rs:3096             Failed(None) => {
src/librustc/middle/resolve.rs:3097                 let mpath = self.idents_to_string(module_path);
src/librustc/middle/resolve.rs:3098                 let mpath = mpath.as_slice();
src/librustc/middle/resolve.rs:3099                 match mpath.rfind(':') {
src/librustc/middle/resolve.rs:3100                     Some(idx) => {
                                    ...
```

[rustc::middle::resolve 3095]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/librustc/middle/resolve.rs#L3095-L3158


* [rustc::middle::resolve 3243] -- (note that even control-flow sensitivity won't save this one)
```
src/librustc/middle/resolve.rs:3243:13: 3271:14 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/librustc/middle/resolve.rs:3243             match search_module.parent_link.clone() {
src/librustc/middle/resolve.rs:3244                 NoParentLink => {
src/librustc/middle/resolve.rs:3245                     // No more parents. This module was unresolved.
src/librustc/middle/resolve.rs:3246                     debug!("(resolving item in lexical scope) unresolved \
src/librustc/middle/resolve.rs:3247                             module");
src/librustc/middle/resolve.rs:3248                     return Failed(None);
                                    ...
```

[rustc::middle::resolve 3243]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/librustc/middle/resolve.rs#L3243-L3271


* [rustc::middle::typeck::check 1748] -- (note that even control-flow sensitivity won't save this one)
```
src/librustc/middle/typeck/check/mod.rs:1748:9: 1759:10 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/librustc/middle/typeck/check/mod.rs:1748         match infer::mk_coercety(self.infcx(),
src/librustc/middle/typeck/check/mod.rs:1749                                  false,
src/librustc/middle/typeck/check/mod.rs:1750                                  infer::ExprAssignable(expr.span),
src/librustc/middle/typeck/check/mod.rs:1751                                  sub,
src/librustc/middle/typeck/check/mod.rs:1752                                  sup) {
src/librustc/middle/typeck/check/mod.rs:1753             Ok(None) => Ok(()),
                                             ...
```

[rustc::middle::typeck::check 1748]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/librustc/middle/typeck/check/mod.rs#L1748-L1759


* [rustc::middle::typeck::infer::error_reporting 175] -- (note that even control-flow sensitivity won't save this one)
```
src/librustc/middle/typeck/infer/error_reporting.rs:175:13: 209:14 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/librustc/middle/typeck/infer/error_reporting.rs:175             match error.clone() {
src/librustc/middle/typeck/infer/error_reporting.rs:176                 ConcreteFailure(origin, sub, sup) => {
src/librustc/middle/typeck/infer/error_reporting.rs:177                     self.report_concrete_failure(origin, sub, sup);
src/librustc/middle/typeck/infer/error_reporting.rs:178                 }
src/librustc/middle/typeck/infer/error_reporting.rs:179 
src/librustc/middle/typeck/infer/error_reporting.rs:180                 ParamBoundFailure(origin, param_ty, sub, sups) => {
                                                        ...
```

[rustc::middle::typeck::infer::error_reporting 175]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/librustc/middle/typeck/infer/error_reporting.rs#L175-L209

* [rustc::middle::typeck::infer::error_reporting 487] -- (note that even control-flow sensitivity won't save this one)
```
src/librustc/middle/typeck/infer/error_reporting.rs:487:9: 784:10 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/librustc/middle/typeck/infer/error_reporting.rs:487         match origin {
src/librustc/middle/typeck/infer/error_reporting.rs:488             infer::Subtype(trace) => {
src/librustc/middle/typeck/infer/error_reporting.rs:489                 let terr = ty::terr_regions_does_not_outlive(sup, sub);
src/librustc/middle/typeck/infer/error_reporting.rs:490                 self.report_and_explain_type_error(trace, &terr);
src/librustc/middle/typeck/infer/error_reporting.rs:491             }
src/librustc/middle/typeck/infer/error_reporting.rs:492             infer::Reborrow(span) => {
                                                        ...
```

[rustc::middle::typeck::infer::error_reporting 487]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/librustc/middle/typeck/infer/error_reporting.rs#L487-784

* [rustc::metadata::decoder 627] -- (note that even control-flow sensitivity won't save this one)
```
src/librustc/metadata/decoder.rs:627:5: 642:6 warning: conflicting match modes, #[warn(match_arms_move_mode_conflict)] on by default
src/librustc/metadata/decoder.rs:627     match decode_inlined_item(cdata, tcx, path, item_doc) {
src/librustc/metadata/decoder.rs:628         Ok(ref ii) => csearch::found(*ii),
src/librustc/metadata/decoder.rs:629         Err(path) => {
src/librustc/metadata/decoder.rs:630             match item_parent_item(item_doc) {
src/librustc/metadata/decoder.rs:631                 Some(did) => {
src/librustc/metadata/decoder.rs:632                     let did = translate_def_id(cdata, did);
                                                                        ...

```

[rustc::metadata::decoder 627]: https://github.com/rust-lang/rust/blob/29f817fa22c906b4bb764e43fb6877a6cde3c07f/src/librustc/metadata/decoder.rs#L627-L642


[RFC PR 210]: https://github.com/pnkfelix/rfcs/blob/fsk-nzdrop-rfc/active/0000-remove-drop-flag-and-zeroing.md
