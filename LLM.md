# CLP(Set) working notes

This document records the state of the `clpset` branch and a conservative route
for developing first-class finite sets in `core.logic`. It is intended as a
handoff for future implementation sessions.

## Sources and scope

The primary source is Dovier, Piazza, Pontelli, and Rossi, "Sets and constraint
logic programming", DOI [10.1145/365151.365169](https://doi.org/10.1145/365151.365169),
published in 2000. Its CLP(SET) fragment is a constraint language over finite
sets with equality, membership (`in`), union, and disjointness (`||`), together
with positive and negative literals. Figure 3 is the equality rewriting
procedure. It represents a non-empty set as a base/tail plus a finite list of
members and repeatedly rewrites equality by selecting members and branching on
whether selected members are equal, removed from one side, or remain in an
open tail. The procedure is nondeterministic because set equality does not give
an ordering for members.

The practical comparison source is Nada Amin's
[clpset-miniKanren](https://github.com/namin/clpset-miniKanren). Its current
`master` and the commit referenced by this branch are both
`4a743c4137453b865258ae0120ea34c604fc8ef7` (`Initial test translation` in this
repository's history). The Scheme source and tests are therefore a useful
executable specification, but the GitHub repository does not have a later HEAD
with the suspected fixes. Its history should be read in order when importing
later semantic patches.

This is a finite-set solver, not a general implementation of arbitrary Clojure
set values. A set term needs to distinguish a known finite member collection
from an open tail variable. The tail is not itself an ordinary Clojure set
member; it denotes the rest of the set.

## Current state

The branch is based on `origin/master` at `6b51197`. Its set-specific work is
the `4a1a5d5` test translation plus the set scaffold in
`src/main/clojure/clojure/core/logic.clj`.

Important status facts:

* The active Scheme-style constraint-store translation and the active set-term
  scaffold were quarantined in comments to restore the namespace. Before that
  repair, compilation failed at `core/logic.clj:890` on unresolved `define`,
  followed by an invalid attempt to take the value of the `llist` macro.
* The ordinary unifier protocol remains active. Its persistent-set dispatch now
  handles fully ground set equality; open-set rewriting is not yet enabled.
  The later library relation block remains inside a `(comment ...)` form.
* A bounded `SetTerm` unifier is now active. It supports extensional equality
  for fully ground `SetTerm` values and ground Clojure sets, including member
  permutation, but rejects open terms until Figure 3 rewriting is ported.
* Closed finite set equality is now relational: `==` can enumerate member
  correspondences when a `SetTerm` has the empty tail and logic variables in
  its members. This is intentionally separate from open-tail rewriting.
* Nested closed sets are routed through the same relational matcher, so logic
  variables inside nested finite sets are covered without enabling open tails.
* The next Fig. 3 implementation must introduce a branching goal layer and a
  residual constraint representation. The existing `cgoal` invocation path
  propagates one state at a time and cannot safely receive a `Choice` as the
  result of a constraint step. Open-set alternatives must therefore branch as
  goals, with residual constraints reified separately.
* The first open-tail case is now implemented: self-membership equality such
  as `q = {q, 1}` rewrites to an open `SetTerm` with member `1` and a fresh tail,
  while retaining a reified `(set _0)` residual constraint. This is a narrow
  Fig. 3 branch, not yet the general equality procedure.
* Repeated self-membership now preserves existing members and reuses the open
  tail. The current `eq-2`-style case is a single normalized branch; it does
  not yet reproduce the reference's five nondeterministic answers. That
  multiplicity remains an explicit compatibility gap.
* Additional narrow rewrites now cover nested self-membership and a shared
  variable appearing as a member on both sides, including `#{q 2} = #{q 1}`.
  These produce structurally correct open terms, but they are not a general
  replacement for Figure 3's member-selection procedure.
* The entire translated Fig. 3 test block in `tests.clj` is inside a
  `(comment ...)` form. These tests are not loaded by `clojure.test`.
* The proposed `subseto` and `!subseto` definitions are also commented out.
* Consequently, the branch does not currently provide usable first-class set
  support, and no Fig. 3 test is currently a repository test that can pass or
  fail.
* With Leiningen installed, `lein test` now passes 438 tests and 695
  assertions, with 0 failures and 0 errors. This validates the executable
  representation/protocol slice without disturbing ordinary core.logic
  behavior.
* The branch now includes the later `b8dad1c`-style `walk-term` implementation
  for `IPersistentSet`. This is required because a persistent set otherwise
  falls through to the generic walker and recursively walks itself forever.

Thus the honest answer to "does the current code work for all current Fig. 3
tests?" is no: the tests are inactive and the implementation is inactive. The
visible expected results are a valuable translation artifact, not a passing
acceptance suite.

## What has been translated

The executable representation slice now defines `SetTerm`, `set-term`,
`set-term?`, `set-term-base`, `set-term-members`, `set-term-tail`, and
`normalize-set` in `core.logic.clj`. `SetTerm` deliberately stores the open
tail separately from an ordered vector of members. Normalization flattens
nested internal set tails, preserves member order, is idempotent for this
representation, and leaves ordinary ground Clojure sets unchanged unless
extra members are explicitly supplied. It does not yet integrate with set
unification, but it now integrates with walking, occurs-check, and term
building; focused tests cover those paths.

Fully ground equality is also executable: the unifier compares the extensional
union of members and ground tails, including permutation-independent equality
with ordinary Clojure sets. Open-tail equality remains intentionally unsupported.

Closed terms with member variables use a lazy matcher in `==` that enumerates
possible member correspondences through core.logic streams, including nested
closed sets. Unequal cardinality fails immediately; open tails still fail until
the residual Fig. 3 constraint is implemented.

The first residual open-tail constraint is `seto`. It validates that a tail is
a set when it becomes known and reifies as `(set tail)`. The current rewrites
handle self-membership, nested self-membership, repeated self-membership
preservation, and one shared-variable cross-side case. General member
selection, open/open tail sharing, and reference-level multiplicity are still
pending.
Repeated self-membership is covered as a preservation regression, but its
reference-level branch multiplicity is still pending.

The branch attempts to port the Scheme representation and equality solver:

* Constraint-store slots corresponding to sets, disequality, non-membership,
  union, disjointness, and symbol constraints.
* `llist`/`lcons` as the internal representation of a set term, with the open
  tail at the end and members preceding it.
* `normalize-set`, tail/member accessors, and the four major equality branches
  in `unify-with-set*`.
* An `IPersistentSet` unification dispatch.
* The reference Fig. 3 examples for equality, followed by membership,
  disequality, union, disjointness, and a few library/implementation tests.

The equality tests translated from the reference are `clpset-run-eq-1` through
`clpset-run-eq-5`. They cover self-membership, repeated equality, variables in
two sets, nested sets, and the same open tail on both sides. The test block
also includes the reference's later regression examples, including the
non-deterministic pair-of-sets case only in the upstream Scheme repository,
not in this branch.

## Idiomatic Clojure decisions and divergences

These choices are reasonable directions, but each needs a small proof-oriented
test before being treated as final:

* The exploratory code uses `lcons`/`llist`, but the executable slice uses an
  explicit `SetTerm` record. This avoids treating ordinary improper lists as
  sets and makes the base/member invariant visible to the type system.
* Use Clojure `IPersistentSet` for user-facing ground sets, while normalizing
  them to the internal open-tail representation for solving. This is idiomatic
  at the API boundary, but nested Clojure sets cannot contain logic variables
  as ordinary members in the same way as the Scheme vector representation;
  equality and reification must define exactly when conversion occurs.
* Use `cond`, `loop`, `recur`, destructuring, and protocol extension instead of
  Scheme's `define`, `let loop`, and vector primitives. This is syntactic
  modernization, not a semantic change.
* Reuse existing `walk`, `bind`, `==`, `conde`, `fresh`, `occurs-check`, and
  `set` names. This is attractive but currently unsafe: `set` already means
  Clojure's set constructor in ordinary code, while the Scheme source uses
  `set` as a set-term constructor. A public API name such as `seto` or a
  dedicated set-term constructor must be chosen deliberately.
* `set-mems` accepts `coll?`, and `normalize-set` accepts Clojure sets. This is
  convenient but broadens the representation contract beyond the paper. Maps,
  vectors, lists, and records need explicit policy rather than accidental
  collection handling.

## Questions and doubts recorded in the code

These are design questions, not TODOs to silently resolve during a translation:

1. Should maps unify with sets, or must set terms be rejected when the other
   side is a map? The current note says this is undecided.
2. What exactly is the empty open-set representation? The Scheme code uses a
   distinct empty set and a false/no-tail result; the Clojure code mixes `()` ,
   Clojure sets, logic variables, and `nil` in `normalize-set`.
3. Is `(llist member tail)` unambiguously a set term in every operation? What
   prevents an ordinary improper list from entering the set solver?
4. What should `set-tail` return for a ground Clojure set, an empty set, a
   symbol, and an lvar? The current implementation relies on normalization but
   does not state the invariant.
5. Does `normalize-set` preserve member order only as an implementation detail,
   or can order affect branch order and therefore observable result order?
6. Are duplicate members removed at normalization time, by unification, or by
   constraints? Clojure sets remove duplicates immediately; open set terms do
   not necessarily do so.
7. Does `occurs-check` traverse the set tail as well as members, including
   nested open sets?
8. Are `walk*`, reification, and constraint reification set-aware? The active
   generic collection walkers are not automatically a proof of the CLP(Set)
   representation's invariants.
9. How should set terms interact with the existing `IPersistentSet` walker added
   in `b8dad1c`? User-facing ground sets and internal open sets may need separate
   protocols.
10. What is the intended public surface: only `==` plus `seto`, or also
    `ino`, `!ino`, `uniono`, `!uniono`, `disjo`, `!disjo`, `subseto`, and
    `!subseto`? Which relations are paper semantics and which are convenience
    relations?
11. What answer multiplicity is acceptable? The reference deliberately exposes
    duplicate and subsumed answers in one test. core.logic normally makes
    result ordering and duplication observable, so this needs an explicit
    policy rather than accidental preservation.
12. Which Clojure versions are supported? The project declares Clojure 1.7;
    implementation choices must not rely on newer collection or reader behavior.

## Conservative commit plan

Keep each step independently reviewable and semantically close to the paper:

1. Add an internal set-term type/predicate and document invariants: finite
   members, one open tail, and the empty-set case. Add representation tests
   only; do not change unification yet.
2. Add normalization, member/tail operations, and occurs/walk/reify support.
  The executable implementation now covers representation, nested-tail
  normalization, walking, occurs-check, building, and ground equality.
  Remaining tests must cover aliased and cyclic cases.
3. Port Figure 3 equality rewriting as a private relation with no other
  constraints. Ground, closed relational, and the first self-membership open
  cases are now covered; the remaining work requires the other member-selection
  branches, open/open tail sharing, and residual constraint cases. Then
  activate only the five
  equality tests, then add the reference
   pair-of-sets nondeterminism regression. Compare normalized answer multisets
   and branch counts with the Scheme oracle.
4. Integrate set unification into the existing unifier and constraint-store
   lifecycle. Run the complete pre-existing core.logic suite after each change.
5. Add membership/non-membership, disequality, union, and disjointness one
   family at a time. Keep each family’s constraint representation and
   reification tests separate.
6. Add convenience relations (`subseto`, etc.) only after their primitives are
   stable. Then add ClojureScript parity if the public API is intended to be
   cross-platform.
7. Remove exploratory Scheme forms and comments only after all active tests and
   invariants are documented. Avoid a broad rewrite while semantic parity is
   still being established.

## Stronger testing strategy

Nada's examples are excellent golden tests, but they mostly assert selected
answer lists. A more comprehensive suite should have several layers:

* **Representation laws:** empty/ground/open round trips; normalization is
  idempotent; `set-mems` and `set-tail` satisfy the stated invariant; nested
  sets remain nested; duplicate ground members have set semantics.
* **Equality laws:** reflexivity, symmetry, ground agreement with Clojure
  equality, permutation independence, empty-set behavior, aliasing, nested
  sets, open-tail versus ground-set cases, and occurs-check rejection.
* **Branch completeness:** the Fig. 3 examples plus the reference
  pair-of-sets-duplicates test. Assert that every expected logical case is
  present, then separately assert whether duplicates/subsumed answers are
  intentional. Do not use only `set` on the result if multiplicity matters.
* **Constraint interaction:** test each primitive against ground truth by
  enumerating small finite universes, including empty sets, singleton sets,
  equal members, nested sets, and open tails. Check positive and negative
  relations, not just reified examples.
* **Cross-product/property tests:** generate small ground sets and terms with
  bounded variables; compare `==`, membership, union, and disjointness against
  a simple finite oracle. Keep generated terms small because nondeterministic
  set equality grows rapidly.
* **Operational tests:** finite termination for ground queries, fair behavior
  for relational queries, no stack overflow in walking/reification, no mutable
  state leaks, and stable behavior under repeated constraint propagation.
* **Compatibility tests:** the complete existing Clojure suite, ClojureScript
  compilation/tests if the feature is shared, Clojure 1.7 compatibility, and
  explicit tests showing ordinary lists/maps/vectors do not get misclassified
  as set terms.

Use the Scheme implementation as a differential oracle for the paper examples,
but do not copy its bugs or its answer-order assumptions without a reason. For
each discrepancy, first classify it as representation, branch ordering,
duplicate-answer policy, or actual logical unsoundness. A useful minimal
acceptance criterion is: no false answers, no missing answers in bounded
ground enumeration, and documented multiplicity/order behavior.

## Immediate next check

The project-compatible Leiningen/Clojure 1.7 baseline is green at 435 tests
and 683 assertions. The first self-membership open-tail rewrite is now
executable. The next slice should port the remaining Figure 3 member-selection
branches, then activate the translated equality examples
(`clpset-run-eq-1`). That test is the cheapest discriminating check for goal
routing, constraint lifecycle, reification, and stream branching. Only after
it passes should the remaining equality examples be activated.