# Narrowed `Any`: closed-class-of-conformers existentials with `A | B`

* Proposal: SE-NNNN (Pitch)
* Authors: [Tao Zhuang][miku1958]
* Review Manager: TBD
* Status: Pitch
* Implementation: prototype on a fork of `swiftlang/swift` at [miku1958/swift][fork-swift], branch [`narrowed-any/phase1-poc`][fork-branch]; see [Implementation status](#implementation-status).
* Pitch thread: [Re-Proposal Type only Unions][thread-72700]
* Prior discussions: [Sum Types, Type Disjunctions and Possible Alternatives][thread-72574], [Disjunctions in types][thread-72668], [Structural Sum Types][thread-67432], [SE-0413 typed throws needs union types][thread-70740]

> **Reading note.** This document deliberately avoids the label *union types* for what it proposes. That label carries strong prior-art meaning from TypeScript and the long line of rejected Swift pitches; reviewers tend to pre-judge on the surface. The feature here is *not* a structural union, *not* a sum / coproduct, and *not* an anonymous enum. It is a **narrowed existential** — `Any` constrained to a closed, declared set of dynamic types. The word "union" only appears below in references to other languages' features and to historical Swift threads.

## Introduction

This proposal adds the spelling `A | B` (and recursively `A | B | C`, `(A | B) | C`, …) for a **closed-class-of-conformers existential**: a value whose dynamic type is statically declared to be one of a closed set of named alternatives. Semantically it is exactly Swift's `Any`, restricted at the source level to a finite list of conforming dynamic types — hence the name *narrowed `Any`*. Layout-equivalent to `any`; no new metadata kind, no new ABI category, no runtime change.

The headline use case is **typed throws across heterogeneous error domains** — the sub-feature [SE-0413] explicitly left for follow-up work. Today's workarounds (wrapper enums, untyped `throws`, custom `Either<A, B>` library types) scale quadratically (`O(N×M)` in API surface × error sources per API) and do not compose across library boundaries. The narrowed-`Any` form replaces all of them with one syntactic construct that composes linearly (`O(N+M)`) and unblocks the rest of the typed-throws story at the same time: exhaustive cross-domain `catch`, generic propagation through `rethrows`-style code, untagged `Codable` for sloppy JSON, and `where T: A | B` constraints.

Beyond errors, the same shape fills the gap that has driven the recurring "closed set of dynamic types" pattern in Swift codebases for years — wrapper enums, `OneOfN<…>` library types, and sealed-protocol scaffolding everyone reinvents. A single anonymous closed-class-of-conformers existential makes those patterns first-class.

The rest of the document explains the syntax, semantics, and type-system rules. § [Motivation](#motivation) makes the cost concrete with a four-way side-by-side of the same multi-source-of-failure API; § [Proposed solution](#proposed-solution) gives the surface; § [Detailed design](#detailed-design) covers the rules; § [Source compatibility](#source-compatibility), § [ABI compatibility](#abi-compatibility), and § [Implications on adoption](#implications-on-adoption) record the binary-compatibility story; § [Implementation status](#implementation-status) lists what is wired in the prototype today; § [Future directions](#future-directions) lists items deliberately deferred from v1.

### Naming

The feature is labelled **narrowed `Any`** throughout, but the name is up for review. The choice matters because it frames how reviewers approach the proposal (and how easily they pattern-match it onto rejected prior art) and because it has to read naturally at *both* the value level (`let v: Int | String`) and the constraint level (`where T: Int | String`, `throws(Int | String)`).

| Name                                     | Reading                                                                                                                                         | Pros                                                                                                                                                                                                                  | Cons                                                                                                                                                                                                |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **narrowed `Any`** *(current)* | "the type `A \| B` is an `Any` narrowed to a closed set of dynamic types"                                                                    | Connects directly to a concept Swift users already know (`Any`). Emphasises the *value-layout* truth: it really is an existential `Any` with a closed conformer set. Foregrounds "this is not a union" upfront. | Less precise once you reach the**constraint** position: `where T: A \| B` is set membership, not narrowing. Two-word term, requires the reader to internalise that `Any` here is the noun. |
| **Closed Type Set**                | "`A \| B` denotes the closed type set `{A, B}`; a value's dynamic type belongs to the set; `where T: A \| B` says T is a member of the set" | Reads naturally at both value*and* constraint level. Aligns with the set-membership semantics. Honest about the closed-class-of-conformers nature. Avoids "union" entirely.                                                       | Three words; risks visual collision with the standard library's `Set<T>` type. Newer term — no existing Swift mental hook.                                                                       |
| **Type Union**                     | "`A \| B` is a type union of `A` and `B`"                                                                                                  | Most direct, what most other languages call this. Lowest friction for users coming from TypeScript / Scala.                                                                                                           | Carries every connotation the*Reading note* above warns about. Reviewers may pattern-match to TS-style structural unions and pre-reject. Precisely what historical pitches were rejected as.      |

This document uses **narrowed `Any`** as the primary term, on the grounds that the value-level reading is the easier mental hook for new users and the constraint-level reading can be expressed with one extra sentence. If review reveals that the constraint reading dominates real-world use (typed-throws-heavy code, generic libraries), **Closed Type Set** is the recommended alternative, as it matches both readings directly.

**Type Union** is listed in the table above *only as a contrast point* — adopting it as the eventual name would re-introduce exactly the prior-art baggage the Reading note above rejects (TypeScript-flavoured pre-judgement, "type-only unions have been rejected before" reflex), so it is not a serious candidate. The two recommended names are **narrowed `Any`** and **Closed Type Set**; the semantics described below are the same whichever of those two wins.

## Anticipated objections

The "type-only unions" line of pitches has been rejected enough times that reviewers reach for the same set of objections on instinct. The table below names each one and gives the response up front, before the reader gets into the design — every objection is addressed in a dedicated section later, but the short answer goes here so review does not stall on a pre-judgement of rejected prior art.

| Objection | Response |
| --- | --- |
| "Swift philosophy doesn't favour structural types." | This is *not* structural. The conformer set is nominal and closed: `A \| B` enumerates the two leaf types by name. The visible interface is the **join**, exposed nominally; no methods are synthesised by structural intersection. See § [The join](#the-join-what-members-are-visible). |
| "Type-only unions have been rejected multiple times." | The rejected designs were either implicitly converting (TypeScript-flavoured) or magic-dispatching (anonymous enums with auto-`.case` synthesis). This proposal does *neither*: leaf injection is the only implicit move, narrowing requires `as?` / `as!`. See § [Cross-shape conversion](#cross-shape-conversion). |
| "It adds a new kind of type." | At the type-system level `A \| B` is a first-class type with its own identity (mangling, witness selection, extension target), and at the runtime level it reuses Swift's existing existential machinery — same `Any`-singleton metadata + `swift_dynamicCast` already shipped for [SE-0309][SE-0309] (`any P`) and [SE-0353][SE-0353] (parameterised existentials). No new metadata kind, no new layout, no new ABI category. See § [Runtime representation](#runtime-representation) and § [ABI compatibility](#abi-compatibility). |
| "Wrapper enums are good enough for typed throws." | True for a single library. Across libraries the wrapper enums don't compose: combining `enum E1 { case a, b }` and `enum E2 { case c, d }` requires a third hand-written `enum E3` that re-cases everything (quadratic). `A \| B \| C` composes linearly. See § [`O(N×M)` → `O(N+M)`](#onm--onm). |
| "TypeScript-style unions cause problems in practice (e.g. erasure)." | This is not TypeScript's union. TS unions are structural and lose nominal identity (`{ kind: "a" } \| { kind: "b" }` is a discriminated record); ours are nominal and over a closed class of conformers. There is no `typeof` narrowing, no implicit subtype upcast across shapes. See § [TypeScript-style structural unions](#typescript-style-structural-unions) for the contrast. |
| "What about ergonomics on the wrapper enum side?" | Conformance synthesis mirrors what wrapper enums already get: untagged `Codable`, exhaustive `switch`, a join-based interface for shared methods. No fewer features than enums, and one extra spelling. See § [Conformance synthesis](#conformance-synthesis-v1-scope). |
| "It bloats the language with another way to spell things." | Anonymous closed-class-of-conformers existentials are missing today; this fills *exactly* the gap that drove the wrapper-enum-of-errors idiom. Either we add the spelling, or every team keeps reinventing wrapper enums for the same shape. See § [Motivation](#motivation). |
| "The `\|` operator clashes with bitwise-or." | Disambiguated by parser context: `\|` in a type position is the type-list separator, in an expression position it is bitwise-or. The same trick `?` already plays for `Int?` versus `expr?`. No source-code break. See § [Issue 1](#issue-1-parsing--in-type-vs-expression-context). |
| "Does adopting this need an OS upgrade / `@available` annotation?" | No. The feature is purely additive at the type-checker / IRGen level; runtime reuses the existing `Any` singleton metadata + `swift_dynamicCast` entry point. Back-deploys to any Swift-supporting OS the project already targets. See § [Implications on adoption](#implications-on-adoption). |

## Motivation

### Swift already has the type the community keeps reinventing

Swift's `Any` is the type whose values may have any dynamic type. The compiler treats `Any` as an existential, opaque box; access happens through `is` / `as?` / `switch case _ as T:`. This is universally understood and predictable.

What Swift does *not* spell, but the community keeps reaching for, is the same shape with a closed list of alternatives. The four current workarounds are:

1. Define a wrapper `enum Result3 { case a(A); case b(B); case c(C) }` per call site.
2. Use a generic library type such as `OneOf3<A, B, C>` — same shape as the wrapper enum, just imported instead of declared per-library, and still positional-discriminator-based (`.first` / `.second` / `.third`).
3. Use `Any` and pay the runtime cost of an open universe, losing exhaustiveness in `switch`, losing typed `throws`, losing safe `as` patterns at compile time.
4. Define a closed protocol that all alternatives conform to (only viable for types you own).

All four are the shape `A | B` with extra ceremony.

### The motivating example: typed throws

[SE-0413] introduced typed throws but explicitly punted on multiple-error throws "until we have a way to spell `A | B`". The community has since produced workarounds — wrapping enums, untyped `throws`, custom `Either<A, B>` shapes — none of which scale past two error types and none of which compose across libraries. Concretely:

```swift
// Wanted (no proposal yet allows this):
func loadUser() throws(NetworkError | DecodingError | AuthError) -> User
```

The wrapper-enum alternative requires every API author to invent `enum LoadUserError { case network(NetworkError); case decoding(DecodingError); case auth(AuthError) }`, every caller to switch through a wrapper that adds nothing, and every consumer of multiple wrapped APIs to invent a third-level wrapper if they want to forward. The cost grows quadratically in API surface; the named entity carries no information.

#### End-to-end: a multi-source-of-failure API, four ways

To make the cost concrete, here is what one realistic API looks like under each available approach. The use case is a session-bootstrap function that hits the network, decodes a payload, and verifies an auth token — all three failure modes are real and the caller usually wants to handle them differently.

**(1) Untyped `throws` (today's idiomatic answer).** Loses every benefit typed throws was supposed to deliver:

```swift
func bootstrapSession() throws -> Session {
    let data = try fetch()                       // throws NetworkError
    let payload = try JSONDecoder().decode(Payload.self, from: data)  // throws DecodingError
    return try verify(payload)                   // throws AuthError
}

// Caller:
do {
    let s = try bootstrapSession()
} catch let e as NetworkError    { ... }
  catch let e as DecodingError   { ... }
  catch let e as AuthError       { ... }
  catch                          { fatalError("unreachable, but the compiler doesn't know") }
```

The exhaustiveness checker cannot rule out the trailing `catch` because `throws` is an open universe. Every consumer pays the same trailing-`catch` cost. Refactoring the body to add a fourth failure mode is silent.

**(2) Wrapper enum.** Restores typing but at quadratic cost:

```swift
enum BootstrapError: Error {
    case network(NetworkError)
    case decoding(DecodingError)
    case auth(AuthError)
}

func bootstrapSession() throws(BootstrapError) -> Session {
    do { let data = try fetch() }  catch { throw .network($0) }
    do { let p    = try decode() } catch { throw .decoding($0) }
    do { return try verify(p) }    catch { throw .auth($0) }
}

// Caller:
do {
    let s = try bootstrapSession()
} catch .network(let e)  { ... }
  catch .decoding(let e) { ... }
  catch .auth(let e)     { ... }
```

The wrapper layer adds no information. It is purely re-casing already nominal errors. Worse: a *consumer* who composes two such APIs ends up writing a *third* wrapper enum to merge `BootstrapError` with some other library's `LoadUserError`. The cost is `O(N×M)` in the number of APIs and the number of error sources per API.

**(3) `OneOf3<A, B, C>` library type.** Same shape, imported instead of authored:

```swift
func bootstrapSession() throws(OneOf3<NetworkError, DecodingError, AuthError>) -> Session { ... }

// Caller:
do {
    let s = try bootstrapSession()
} catch .first(let e)  { ... }
  catch .second(let e) { ... }
  catch .third(let e)  { ... }
```

Same content, different mechanical wrapper. The discriminator names `.first` / `.second` / `.third` are arbitrary positional indices that drift the moment someone adds a new error type — adding a fourth source of failure forces a switch from `OneOf3` to `OneOf4`, which is a different (unrelated) type with its own discriminator names, so every existing call site has to be rewritten. Composing across libraries inherits the same friction: two libraries that each expose a `OneOf3<…>` over different leaf sets cannot be merged without a hand-rolled wrapper that re-cases both sides.

**(4) Narrowed `Any` (this proposal).** No discriminator, no wrapper, exhaustive switch, composable across libraries:

```swift
func bootstrapSession() throws(NetworkError | DecodingError | AuthError) -> Session {
    let data = try fetch()                       // NetworkError flows through
    let p    = try JSONDecoder().decode(...)     // DecodingError flows through
    return try verify(p)                         // AuthError flows through
}

// Caller (exhaustive — no trailing catch):
do {
    let s = try bootstrapSession()
} catch let e as NetworkError  { ... }
  catch let e as DecodingError { ... }
  catch let e as AuthError     { ... }
// switch over the thrown closed set is exhaustive — no `default` needed
```

No wrapper enum, no positional discriminator, full exhaustiveness, no information lost across library boundaries. Adding a fourth failure mode requires `throws(NetworkError | DecodingError | AuthError | RateLimitError)` and the existing call sites get a *compile-time* warning that their `catch` set no longer covers the closed leaf set.

The two forms compose: a user can write `catch let e as NetworkError { ... }` for one leaf and individual `catch .malformed { ... }` arms for another. The exhaustiveness checker treats each leaf independently — `IsPattern` coverage *or* full enum-case coverage — and accepts the do-catch as soon as every leaf is exhausted by either form.

This composes transparently with `async`: a function declared `async throws(A | B)` awaited inside `do { try await … } catch .case { … }` dispatches identically — the await suspension boundary is invisible to the catch-arm machinery, and an exhaustive set of catch arms over the union of leaf cases means the enclosing function need not itself be `throws` or supply a `default` arm.

#### `O(N×M)` → `O(N+M)`

The named-entity tax noted above has a specific mechanism. Every wrapper enum, every `OneOfN<…>` alias, carries a discriminator (`.network(_)` / `.first(_)` / `.a(_)`) that the user invents, the call site immediately destructures, and the linker mangles separately for each library. The discriminator becomes part of the type identity, and so the named entity does not compose across library boundaries: combining `enum E1 { case a, b }` with `enum E2 { case c, d }` requires a third hand-written wrapper that re-cases everything, and the cost grows quadratically in API surface.

Narrowed `Any` removes the discriminator entirely. `A | B` is not a wrapper around `A` and `B`; it is a refinement of `Any`. There is nothing to drift, nothing to invent a name for, and nothing for two libraries to disagree about. Two libraries that both export `(NetworkError | DecodingError)` use the same type — different libraries' alternations compose automatically because the type identity is the *set of leaves*, not a wrapping name.

The `O(N×M)` blow-up of approaches (2) and (3) becomes `O(N+M)`. That is the number that drove this proposal.

### Why this is the right shape for Swift

Swift has spent a decade resisting structural unions for good reasons: they conflict with nominal typing, lose dispatch coherence, and historically push reviewers towards TypeScript's runtime-typeof model that does not fit a statically-typed nominal language.

The right shape is therefore an *anonymous* closed-class-of-conformers type — the conformer set is named (each leaf is a real, nominal Swift type), but the type that bundles them is not. Swift already has all the infrastructure for this: `any P` is an existential, `any` carries a witness table, `is` / `as` open the box, `switch case let _ as T:` is the dispatch pattern. All of that machinery applies, unchanged, to a closed-class-of-conformers existential — the only addition is that **the compiler computes the members visible on the box from the *least upper bound (LUB)* of the listed alternatives instead of from a single protocol's requirements**. This LUB computation — the smallest common supertype every leaf conforms to — is the same notion Swift's type-checker already uses to pick an inferred type for `let x = cond ? a : b` today; here it is applied to a closed user-spelled list of alternatives instead of to the two arms of an `if` / `?:`.

This proposal does *not* import the broader semantics that surface-similar features in other languages have shipped — Scala 3 union types' untagged-sum / unboxed / commutative semantics, TypeScript / Python / Crystal's structural assignability, F# / Rust's discriminated-union nominal-tag-and-payload story. Each of those would silently change Codable order, witness-table selection, or mangled identity in ways Swift's nominally-typed surface cannot afford. § [Comparison with how other languages spell the same idea](#comparison-with-how-other-languages-spell-the-same-idea) lays out the contrasts and § [Spelling is identity](#spelling-is-identity) explains why the proposal commits to the conservative posture instead.

## Proposed solution

### The type

`A | B` denotes a closed-class-of-conformers existential whose dynamic type is statically declared to be one of `A` or `B`. The form recurses: `A | B | C`, `(A | B) | C`, `A | (B | C)`, `Int | (Double | String) | Bool`, etc. Both leaves and intermediate sub-alternations may be any concrete or generic Swift type — classes, structs, enums, tuples, function types, and existing existentials — as long as each leaf is a *fully* concrete type at the point the alternation is written.

```swift
let v: Int | String      = 7              // one leaf injection
let v: Int | String      = "hi"           // another
let f: (Int | String) -> Void             // function-parameter position
func parse() throws(NetworkError | DecodingError) -> Payload
extension Array where Element == Int | String { ... }
```

### The "as `any`" framing

Operationally, `A | B` is `any (A or B)`. The "as `any`" framing — that the value is a black-box existential whose interface is the *join* of the alternatives — is the anchor for everything that follows:

* The runtime layout is identical to `any` (24-byte inline value buffer + metadata pointer).
* Member access goes through the join's witness table, not through structural intersection of A's and B's members.
* `is` / `as` / `switch case _ as T:` open the box exactly as they do today.
* `Codable`, `Equatable`, `Hashable`, … synthesise via the same mechanism Swift already uses for `any P`: the existential is the witness, dispatch reaches into the leaf via the closed-class-of-conformers table.

The user-visible difference from `any P` is one axis only: the closed conformer set is *enumerated by name* instead of *computed from a single protocol*. Every other behaviour is inherited.

### The join: what members are visible

The *join* of a narrowed-`Any` is the smallest common base type — the **least upper bound (LUB)** of the alternatives in Swift's type lattice, computed from the linearisation of each alternative. (This is the same lattice notion type-checkers across many statically-typed languages use to assign a type to a conditional expression; calling it "join" follows the type-theory convention. § [Comparison with how other languages spell the same idea](#comparison-with-how-other-languages-spell-the-same-idea) places this proposal's choices among the various surface-similar features other languages have shipped.) In Swift terms:

* The visible **members** of `A | B` are the members of `join(A, B)` — the most-derived class or protocol that both `A` and `B` are subtypes of.
* The **conformance** of `A | B` to a protocol `P` holds iff `A: P` and `B: P`.
* `A | B` is a subtype of any non-narrowed `T` such that both `A: T` and `B: T`.
* `A` and `B` are each subtypes of `A | B`.

These join-derived members and conformances are the **only** source of behaviour for `A | B` in v1, because v1 rejects user-written extensions on a narrowed-`Any` target (see [Conformance synthesis (v1 scope)](#conformance-synthesis-v1-scope)). The follow-up [§ Extending a narrowed-`Any` directly](#extending-a-narrowed-any-directly) lifts that restriction; in that follow-up the join-derived behaviour acts as a *fallback* — user-declared methods or conformances on `A | B` itself take priority, the way concrete-type witnesses already shadow protocol-extension default implementations in Swift today.

```swift
extension Int:    CustomStringConvertible { ... }
extension String: CustomStringConvertible { ... }

let v: Int | String = "hi"

// Direct join-member access — `description` is a CustomStringConvertible member
// and both Int and String conform, so it is in the join and dispatches through
// the synthesised witness:
print(v.description)                 // OK: dispatches via the join's CustomStringConvertible witness
print("v=\(v)")                      // OK: string interpolation goes through the same witness

// Generic-position dispatch with the same join protocol — works the same way:
func describe<T: CustomStringConvertible>(_ x: T) { print(x) }
describe(v)                          // OK: Int | String conforms to CustomStringConvertible via the join

// Leaf-only methods — rejected at compile time, because they are not in the
// join. To use a leaf-specific method, narrow first with `as?`:
// v.append("x")                     // error: 'append' is on String, not on every leaf
if let s = v as? String { s.append("x") }   // OK after explicit narrow
```

This makes narrowed `Any` **strictly more useful than open `Any`** — it carries the witnesses for whatever `A` and `B` share, callable directly through the join's witness tables — but **strictly less surprising than TypeScript-style structural unions** — leaf-only methods (those not in the join) are never magically synthesised, and there is no implicit narrowing through structural intersection or runtime `typeof` checks. Reaching a leaf-only method always requires explicit narrowing with `as?`, the same way Swift's `any P` requires opening the existential to call leaf-specific behaviour.

**v1 prototype status.** The example above shows the *design* — per-witness dispatch through the join is the commitment described in [§ Conformance synthesis (v1 scope)](#conformance-synthesis-v1-scope), and the v1 review surface is that design. The current prototype ships only the self-conforming-protocol synthesis (`Error`, marker protocols); for protocols with method requirements (`Hashable`, `Equatable`, `Comparable`, `CustomStringConvertible`) the synthesis is **deferred from v1** with an explicit `as! any P` escape hatch — see [§ Conformance synthesis (v1 scope)](#conformance-synthesis-v1-scope) for the rationale and [Future directions § Per-narrowed-Any witness emission](#per-narrowed-any-witness-emission) for the follow-up that completes the example as shown.

The join is computed lazily on first access and cached as a field on `TypeBase`. First access walks each leaf's protocol-conformance list and class-hierarchy chain, intersecting and finding the LCA; subsequent access is `O(1)`. The constituent lookups (`Module::lookupConformance`, `ClassDecl::getSuperclassDecl`) are themselves already memoised by Swift's `TypeChecker` / `ASTContext`, so even cold-start is bounded by a handful of hash queries.

### Pattern matching is exhaustive

Because the type set is closed and known to the compiler, `switch` over `A | B` is exhaustive iff every alternative is matched:

```swift
let v: Int | String = 7
switch v {
case let n as Int:    print("int: \(n)")
case let s as String: print("str: \(s)")
}                                    // exhaustive — no default needed
```

`as` patterns inside `case` clauses are how each leaf is named. The compiler tracks the closed leaf set and rejects a missing arm with a diagnostic that lists the uncovered leaves. Adding a fourth alternative to a `throws(A | B | C | D)` declaration produces a "switch must be exhaustive" warning at every existing call site that previously covered `A | B | C` — *visible*, not silent.

### The depth-1 principle: structurally nested, behaviourally flat

`(A | B) | C` is a *different type* from `A | B | C` (see [Spelling is identity](#spelling-is-identity) below for why). But for *behaviour* — pattern matching, `as?`, exhaustiveness, leaf injection — the inner and outer alternations collapse into one flat closed set:

```swift
let v: (Int | String) | Bool = 7

switch v {
case let n as Int:    ...     // depth-1 leaf, even though Int sits inside (Int | String)
case let s as String: ...
case let b as Bool:   ...
}                              // exhaustive — three leaves, not "Int | String" + "Bool"
```

The user types `Int | String | Bool` and the compiler computes the same flat leaf set `{Int, String, Bool}`; they type `(Int | String) | Bool` and the compiler computes `{Int, String, Bool}` — same set, different *spellings*. The two spellings differ in identity (mangling, Codable order, witness selection — see [Spelling is identity](#spelling-is-identity)), but neither requires the user to first `as?`-narrow to `Int | String` before reaching `Int`. This is the same experience `Int??` already provides on top of `Optional`.

IDE completion follows the depth-1 rule: a switch on `(A | B) | C` with no body suggests `case let _ as A`, `case let _ as B`, `case let _ as C` — three flat arms, not "narrow first, then expand".

<a id="real-world-buildeither-example"></a>
A real-world example of structurally-nested-behaviourally-flat already in shipping Swift: result-builder `if / else if / else` chains. SwiftUI's `ViewBuilder` produces `_ConditionalContent<_ConditionalContent<A, B>, C>` for a three-arm chain — depth-2 nesting at the type-identity level (each `else` introduces another wrapper layer), but pattern-matching on the resulting view is flat (`Mirror` and SwiftUI's own dispatch walk the deep set). Replacing `_ConditionalContent<T, F>` with `T | F` (a [Future-direction follow-up](#result-builders-simplifying-buildeither-and-the-_conditionalcontent-ladder)) collapses the wrapper struct but inherits the same depth-1 principle for free: `A | B | C` (flat) and `(A | B) | C` (depth-2) are different identities, but a `switch` over either still walks `{A, B, C}` directly.

### Spelling is identity

The single design rule that drives most of the proposal's surface:

> Two narrowed-`Any` types with the same leaf set but a different *written spelling* (different leaf order, different parenthesisation/nesting) are **different types**. They are not normalised, not implicitly convertible, and not interchangeable.

So `Int | String` ≠ `String | Int` (different leaf order); `(Int | String) | Bool` ≠ `Int | String | Bool` ≠ `Int | (String | Bool)` (different *nesting* — the parenthesised sub-alternation becomes its own depth-2 leaf, so the outer leaf set is `{Int|String, Bool}` rather than the flat `{Int, String, Bool}`). Each is its own type with its own mangled name.

The parens that matter here are the ones **inside an alternation**, where they introduce nesting. Parens that are part of *another* type-level construct — the function-type parameter list `(T) -> U`, the tuple type `(A, B)`, or top-level redundant grouping `let v: (Int | String) = 5` (which Swift already collapses to `let v: Int | String`) — do *not* count as "alternation nesting" and do *not* change the alternation's identity. So `let g: (Int | String) -> Void` has parameter type `Int | String` (the function-type parens are syntax, not nesting); only `(Int | String) | Bool` and similar shapes — where the parens sit *inside* an outer `|` — invoke the depth-2 rule.

Why: changing the spelling is observable.

| User-visible behaviour                           | Drives off                                                                      | Consequence of normalising silently                                                                                        |
| ------------------------------------------------ | ------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Codable encode/decode try-order                  | declaration order of the alternation                                            | a "wider" auto-conversion silently flips encoder behaviour — a JSON `42` round-trips as `Int` under `Int \| Double`, as `Double` under `Double \| Int`                                          |
| Switch-completion suggestion order               | declaration order                                                               | IDE arm ordering shifts under the user's feet                                                                              |
| Mangled name (binary identity)                   | spelling                                                                        | adding a leaf "for free" silently breaks ABI                                                                               |
| Protocol-witness selection at extension dispatch | the type that the `extension where Element == ...` clause was written against | `extension Array where Element == Int \| String` does not apply to `[String \| Int]` even though leaf sets are identical |

Forcing the user to write `as` when changing spelling makes the reshape *visible*. The cost — one keyword — pays for itself the first time someone reads the code and asks why the encoder behaviour changed. § [Cross-shape conversion](#cross-shape-conversion) covers the explicit `as` rules; § [Compile-time check](#compile-time-check) covers the algorithm; but the rule itself is one sentence: *spelling is identity*. References below to "synthesised behaviour changes when the spelling changes" all chain back to this paragraph.

#### What is *not* normalised

* Order-sensitivity: `Int | String` ≢ `String | Int`. Not a bug; the spelling is the lever the user holds for Codable and witness behaviour.
* No idempotence collapse: `T | T` is allowed and stays `T | T`. The compiler does not warn (no heuristic — if the user wrote it, they wrote it). It behaves indistinguishably from `T` at runtime, but its identity is `T | T`. § [Issue 4](#issue-4-overlapping-types) covers this.
* No protocol/super-class collapse: `Cat | Animal` (where `Cat: Animal`) is allowed and stays as written. The runtime cast `value as Animal` succeeds on either leaf; the spelling does not change.
* No optional collapse: `T | T?` stays as written. § [Issue 3](#issue-3-interaction-with-optionalt-and-t) covers the parsing precedence between `T?` and `T | nil`.

These are *not* warning categories. The static compiler is silent on `T | T`, `Cat | Animal`, `T | T?`. The size warning (§ [Issue 7](#issue-7-large-or-deeply-nested-narrowed-any)) is the one structural diagnostic that fires, and it is purely about cognitive load on large alternations, not about meaning.

### Source-level surface

The minimum reachable shape is two leaves separated by `|`. Parenthesisation is required only where ambiguity with another type-level construct would arise:

```swift
// Bare alternation — fine in any unambiguous type position:
let v: Int | String
let arr: [Int | String]
let dict: [String: Int | Bool]
func f<T: Int | String>(_: T)
extension Array where Element == Int | String { ... }
throws(NetworkError | DecodingError)

// Parens required where the surrounding context creates ambiguity:
let g: (Int | String) -> Void               // legal — function-type parens; param is `Int | String`.
                                             // The parens are needed because bare `A | B -> C` is ambiguous
                                             // between `A | (B -> C)` and `(A | B) -> C`.
let h: ((Int | String), Int)                 // parens around the alternation when the
                                             // tuple element is the alternation itself

// Parens NOT required when a `:` label or other delimiter already
// closes the type context:
let g: (_ value: Int | String) -> Void       // OK: `:` ends the type before `,`/`)`
struct S { let x: Int | String }             // OK: `:` then `=` / newline closes
```

The parenthesisation rule is *the only one needed for v1*. § [Issue 2](#issue-2-function-type-parenthesization) covers the precise list of positions where parens are required; § [Grammar](#grammar-ebnf) gives the EBNF.

### Runtime representation

A narrowed-`Any` value uses the **`Any`-singleton metadata pointer** at runtime — the same metadata Swift's existing `Any` has used since 1.0. The value is a 24-byte inline value buffer plus a 1-word type-metadata pointer; value-witness pointers reach through the metadata. No new metadata kind, no new layout, no new ABI category, no new runtime entry point.

This is the load-bearing observation for [Implications on adoption](#implications-on-adoption): the feature is purely additive at the language level, and code targeting an older Swift runtime continues to run correctly when re-compiled with a newer toolchain that supports `A | B`. § [Implications on adoption](#implications-on-adoption) spells out the back-deployment story.

## Detailed design

### Subtyping lattice

The subtype rules of `A | B` against non-narrowed types are entirely lattice-driven and consistent with what `any P` already does:

* `A` is a subtype of `A | B`. (Leaf injection — see [Implicit conversion](#implicit-conversion).)
* `B` is a subtype of `A | B`. (Same.)
* For any type `C` that is *not* itself a narrowed-`Any`, `A | B <: C` iff `A <: C` and `B <: C`.

The "not itself a narrowed-`Any`" carve-out matters: implicit conversion from one narrowed-`Any` spelling to *another* narrowed-`Any` spelling is **rejected by design** (see [Spelling is identity](#spelling-is-identity)). Cross-spelling conversion needs explicit `as` — § [Cross-shape conversion](#cross-shape-conversion) covers the rules.

Generic containers do *not* covary in their element type:

* `Array<A | B>` is *not* a subtype of `Array<A>`. `Array` is invariant; changing the element type changes the mangled name.
* Extension dispatch on a generic with a narrowed-`Any` element constraint applies to receivers whose generic argument matches the constraint exactly: `extension Array where Element == Int | String { … }` matches `[Int | String]`. Leaf-typed receivers (`[Int]`, `[String]`) and cross-spelling receivers (`[String | Int]`) need an **explicit cast** in v1 — the compiler emits a fix-it that inserts `(receiver as [A | B])` for the leaf case and `(receiver as [A | B])` for the cross-spelling case (one keystroke each). The implicit lift — accepting a leaf-typed receiver without a cast — is a deferred follow-up; the v1 ergonomic story is "fix-it makes the cast trivial". See [Per-element leaf injection at the extension boundary](#per-element-leaf-injection-at-the-extension-boundary) for why the implicit form is harder than it looks (per-element layout differs between `[Int]` and `[Int | String]`, requiring a runtime element-wrap conversion). § [Containers and extensions](#containers-and-extensions) covers all three axes.

### Implicit conversion

The **only** implicit conversion in this proposal is **leaf-introduction at the value level**: a leaf-typed value may be assigned to a narrowed-`Any` declaration that lists that leaf as one of its alternatives.

```swift
let p: Int = 7
let q: Int | String          = p   // implicit (value level)
let r: String | Int          = p   // implicit (still introducing the leaf Int)
let s: (Int | Double) | Bool = p   // implicit (Int appears recursively)
```

The container-axis analogue — a leaf-typed `[Int]` reaching `extension Array where Element == Int | String` without writing an `as` cast — is a natural extension of this rule, but ships in v1 as **explicit-cast-with-fix-it** rather than implicit, because the per-element runtime layout of `[Int]` and `[Int | String]` differs (8-byte raw `Int` slots vs. 32-byte `Any`-singleton existential slots — 24-byte inline value buffer + 1-word metadata pointer) and the conversion requires a per-element wrapping pass; element stride grows 4× and a fresh buffer must be allocated. See [Containers and extensions](#containers-and-extensions) for the v1 ergonomics and [Per-element leaf injection at the extension boundary](#per-element-leaf-injection-at-the-extension-boundary) for the deferred implicit form.

Every other narrowed-`Any` ↔ narrowed-`Any` conversion **requires explicit `as` / `as?` / `as!`**, even when the relationship is provably safe — see [Spelling is identity](#spelling-is-identity).

### Cross-shape conversion

#### The user-facing rule for `as` / `as?` / `as!`

| Relationship between source and target                                                                                                                              | Operator          | Compile?        | What happens                                                                                                                                                                                                                                                                     |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Every leaf in the source also appears in the target (recursively expanding both) — i.e.`leaves(source) ⊆ leaves(target)` with per-leaf class/protocol subtyping | `as`            | yes             | Always succeeds. The value is rewrapped under the target's spelling without changing the underlying dynamic value.                                                                                                                                                               |
| Every leaf in the target appears in the source, but not vice versa,*or* partial overlap                                                                           | `as?` / `as!` | yes             | At runtime, look at the actual value: if its dynamic type matches the target, return the rewrapped value; otherwise return `nil` (or trap, for `as!`).                                                                                                                       |
| The two have no leaves in common at all                                                                                                                             | any               | **error** | Cast can never succeed for any value, so the compiler refuses (in*all three* forms — `as`, `as?`, `as!`). Stricter than current Swift's `5 as? String`, which is a warning; here the closed leaf set makes "no type in common" a static fact, not a runtime question. |

In one sentence each:

* **`as`** compiles when the cast is guaranteed to succeed for every possible source value — the source's leaf set is a subset of the target's.
* **`as?` / `as!`** compile when the cast might succeed for at least some source values. The runtime, not the compiler, decides via the value's dynamic type. They are **the form you reach for when plain `as` would not compile** — using them on a cast that is already total triggers a warning (see [Cast-feasibility warnings](#cast-feasibility-warnings) below).

This is exactly Swift's existing convention for `as` versus `as?` / `as!` against class hierarchies and protocol existentials — the only thing new is that "type relationship" generalises to walking the recursive structure of narrowed-`Any` types.

#### Examples

```swift
// `as` — total, compile-time:
let a: Int | String = 7
let b: String | Int = a as String | Int                // same leaves, different spelling
let c: (Int | Double) | String = 1.5
let d: Int | Double | String   = c as Int | Double | String   // regrouped
let g: Int | Double = 7
let h: Int | Double | String = g as Int | Double | String     // grow with new leaf

// `as?` / `as!` — partial, runtime:
let i: (Int | Double) | String = "hi"
let j: (Double | String)? = i as? Double | String      // .some("hi") at runtime
let k: (Int | Double) | String = 7
let l: (Double | String)? = k as? Double | String      // nil — k holds Int, not in target
let m: Int | Double = 1.5
if let n: Int | String = m as? Int | String {
    // not entered: m's dynamic type is Double, has no home in Int | String
    _ = n
}

// Compile errors:
let o: Int | Double = 7
o as String | Bool         // error: leaf sets disjoint (no shared leaf)
o as? String | Bool        // error: still disjoint, partial form does not help
o as! String | Bool        // error: still disjoint, would trap unconditionally
o as String | Int          // error: partial overlap — `as` requires every source leaf
                           //        fit; Double has no home in `String | Int`. Use
                           //        `as?` / `as!` for the runtime-decide form, or
                           //        narrow the source first.
```

#### Diagnostic and fix-it for implicit cross-shape

Implicit cross-shape conversion is a hard error, but the compiler turns the rejection into an actionable diagnostic by computing the leaf-set relation and offering the right cast operator as a fix-it:

```swift
let a: Int | String = 7
let b: String | Int = a    // error: implicit cross-shape conversion not allowed
                           //        fix-it: insert ` as String | Int` after `a`
```

| Leaf-set relation (with per-leaf subtyping)               | Fix-it offered                                                                                         |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `leaves(source) ⊆ leaves(target)` (different spelling) | Insert ` as T2` — total cast.                                                                       |
| Partial overlap                                           | Insert ` as? T2` (primary, safe form); secondary fix-it offers ` as! T2`.                          |
| Disjoint                                                  | No fix-it. The diagnostic lists the actual leaves on each side so the user can fix the*target* type. |

The same logic fires anywhere implicit conversion would have applied — assignment, parameter passing, `return` from a function whose return type is `T2`, generic-argument substitution.

#### Cast-feasibility warnings

Where the closed leaf set makes the cast result statically determined and the cast is *not* outright disjoint, the compiler emits a warning rather than waiting for the runtime to give the only possible answer:

* **Same-spelling cast** — `v: Int | String; v as Int | String` (or `as?`, or `as!`) is a no-op in every form. The `as?` / `as!` cases are covered by Swift's existing "conditional cast from `T` to same type `T` always succeeds" / "forced cast of `T` to same type has no effect" warnings (which already fix-it to drop the cast). Plain `as` of a value to its own type is silent under existing Swift convention; the proposal does not introduce a new warning here.
* **Subset cast with different spelling** — `v: Int | String; v as? String | Int` (or `as!`). The static `as` form does real work (rewraps under the target's spelling, which changes Codable order, mangled name, etc.); `as?` / `as!` are dead weight. Warning: "always succeeds; consider using `as`"; fix-it: drop the `?` / `!`.

Disjoint leaf sets are *not* in this category — `as` / `as?` / `as!` are all hard errors when the leaf sets share no types in common (per the rule table above). `is` against disjoint leaf sets stays a warning, matching Swift's existing convention for statically-false `is` checks (the result is still a well-typed boolean false).

#### Compile-time check

Type-checking a narrowed-`Any` cast walks six steps:

1. **Recursive leaf-set extraction.** Compute `leaves(T)` for both sides (`leaves((Int | String) | Bool) = {Int, String, Bool}`). Tuple, array, dictionary, optional and other nominal containers stop the recursion — the leaf set is over narrowed-`Any` structure only, not generic instantiation.
2. **Per-leaf subtype walk** (not name equality). A source leaf `Dog` matches a target leaf `Animal` if `Dog: Animal`; a source leaf `MyError` matches a target leaf `Error` if `MyError: Error`. The check walks the class hierarchy and protocol-conformance graph for each candidate pair, hand-off to the existing type-checker queries Swift already runs for plain class casts.
3. **Set-relation classification.** Subset → `as` valid; overlap → `as?` / `as!` valid; disjoint → error in any form.
4. **Spelling-preserved rewrap emission.** Even when `as` is valid, the source and target spellings may differ (e.g. `(Int | String) | Bool` ≢ `Int | String | Bool` even with the same leaf set). The SIL emits a type-level relabeling so downstream operations dispatch through the target's witness tables; runtime layout is identical because narrowed-`Any` reuses `Any`-singleton metadata, but the compile-time SIL must still re-bind to the target type.
5. **Cast-feasibility warnings.** Per [Cast-feasibility warnings](#cast-feasibility-warnings) above.
6. **Closed-class table emission.** Each narrowed-`Any` declaration site emits a small static table listing the metadata pointer of each leaf. `as?` / `is` against a narrowed-`Any` target consults this table; `switch` exhaustiveness uses it to decide which arms cover the closed set. The table is per-declaration-site, with linker-coalesce-friendly mangling, and is the only persisted artefact in user binaries — no runtime extension.

The naive reading of these steps would be `O(n × m)` per cast site. The actual algorithm sorts each leaf set once by mangling (cached per-type on `TypeBase`), then a single merge walk classifies subset / overlap / disjoint in `O(n + m)`. The classification result for each `(sorted_L, sorted_R)` pair is itself memoised, so repeated cast sites of the same leaf-set shape are `O(1)`. With [Issue 7](#issue-7-large-or-deeply-nested-narrowed-any)'s `≤ 8`-leaf cap, even cold-cache worst-case is dozens of operations per site.

The algorithm is elementary — sorted-leaf-set intersection plus per-leaf class / protocol subtype walks for the "shared leaves" tally. The same closed-leaf-set reasoning underpins this proposal's *switch exhaustiveness* check, which is structurally a [Maranget-style pattern usefulness algorithm][maranget] (used by OCaml, Rust, Haskell, F# for exhaustiveness checking) specialised to closed-conformer existentials; cast feasibility is a simpler set-relation classification on the same closed leaf sets. § [Alternatives considered § Prior art: cast-feasibility algorithms](#prior-art-cast-feasibility-algorithms) places both checks in the broader taxonomy.

#### Runtime semantics for `as?`

`v as? T` operates on the **value itself**, not on the static type. At runtime:

1. Read the dynamic type of `v` from its existential metadata pointer (a single load — same machinery as today's `any P`).
2. Check whether that dynamic type is one the target declares. If yes, the value is rewrapped under the target's spelling (the underlying value buffer is unchanged, only the static type the source-level binding sees is updated). If no, return `nil`.

The static type of `v` does not enter the runtime decision — only the dynamic type does. So a value declared `(Int | Double) | String` but currently holding a `Double` succeeds in `as? Double | String` regardless of how the source-level binding was spelled: the runtime reads the dynamic `Double`, sees that the target lists `Double` as a leaf, and rewraps. The static narrowing on `v` plays no part.

`as!` is `as?` followed by force-unwrap. Standard Swift convention.

The runtime cost is one metadata-pointer comparison plus the rebox — roughly the same as `as? P` against a protocol existential today.

### Containers and extensions

`Array<A | B>` is *not* a subtype of `Array<A>` (Array is invariant). Extension dispatch on a generic constrained by `where Element == narrowed-Any` matches receivers whose generic argument is exactly that narrowed-`Any` spelling. Leaf-typed and cross-spelling receivers need an **explicit cast** in v1, both with a one-keystroke fix-it.

```swift
extension Array where Element == String | Int {
    func summary() -> String { ... }
}

let ys:  [String | Int] = ["a", 7, "b"]
let zs:  [Int | String] = [1, "a"]                 // same leaves, different spelling
let xs:  [String]       = ["a", "b"]               // single leaf, narrowed-`Any` is wider

ys.summary()                                       // OK: same spelling

xs.summary()                                       // error: leaf-typed receiver — fix-it: ` as [String | Int]`
(xs as [String | Int]).summary()                   // OK: explicit lift, walks elements wrapping each as Any-singleton

zs.summary()                                       // error: cross-spelling — fix-it: ` as [String | Int]`
(zs as [String | Int]).summary()                   // OK: explicit reshape (runtime-free relabel)

extension Array where Element == String {
    func leafOnly() -> String { ... }
}

xs.leafOnly()                                      // OK
ys.leafOnly()                                      // error: `[String | Int]` is not `[String]`
(ys as? [String])?.leafOnly()                      // OK: explicit `as?`, fails if any
                                                   //     element is Int
```

The three axes of dispatch:

* **Leaf injection (explicit in v1, with fix-it)** — `[A]` reaching an extension on `[A | B]` needs `as [A | B]`. The cast is *not* runtime-free: `[Int]`'s element stride is 8 bytes (raw `Int`), `[Int | String]`'s element stride is 32 bytes (24-byte inline value buffer + 1-word `Any`-singleton metadata pointer — same shape as a value-level `let v: Int | String = 7`). The cast walks the array wrapping each element, allocates a 4×-larger buffer, and releases the original. **O(N) time, ~4× transient memory peak** for `Int → Int | String`; ratio depends on the leaf's stride relative to 32 bytes. The fix-it makes the cast a one-keystroke fix; the *implicit* form is a deferred follow-up — see [Per-element leaf injection at the extension boundary](#per-element-leaf-injection-at-the-extension-boundary) for the design and the implementation cost.
* **Narrowed → leaf (explicit, partial)** — `[A | B]` reaching an extension on `[A]` needs `as? [A]` because the runtime might hold a `B` element. This is the dual direction; same shape as Swift's current `[Animal] as? [Dog]`.
* **Cross-spelling (explicit, total, runtime-free)** — `[B | A]` reaching an extension on `[A | B]` needs `as [A | B]` to reshape the spelling. Both arrays have bit-identical memory layout (each element is narrowed-`Any` using the `Any`-singleton metadata regardless of leaf order), so the cast is a SIL-level type relabel — no element walk, no buffer reallocation. Per [Spelling is identity](#spelling-is-identity), Codable encoding order, witness selection, and mangling all follow the new spelling. The cross-spelling axis can be relaxed by the [Order-insensitive marker in `where` clauses](#order-insensitive-marker-in-where-clauses) future direction.

In short: cross-spelling is a free relabel (O(1)); leaf injection is an O(N) wrap with 4× transient memory peak; narrowed → leaf is an O(N) walk with conditional success. All three are explicit-`as` / `as?` in v1. The leaf-injection diagnostic carries an auto-fix-it that inserts the cast; the cross-spelling diagnostic currently doesn't (separate diagnostic-quality follow-up; see [Implementation status](#implementation-status)).

**Most-specific-wins on overlap.** When multiple `where Element == …` extensions overlap, Swift's existing extension-dispatch rule picks the most-specific one based on the receiver's static generic argument. So if both `extension Array where Element == Int { … }` and `extension Array where Element == Int | String { … }` are in scope, an `xs: [Int]` receiver calls the `Element == Int` version directly — same as Swift's existing class-hierarchy override resolution. The leaf-injection lift documented above (and its v1 explicit-cast surface) only kicks in for methods that the *more-specific* extension does not provide. This is the container-axis analogue of the receiver-static-type-priority rule documented for value-axis dispatch in [§ Future directions § Extending a narrowed-`Any` directly](#ext-rule-leaf-reach).

**Library-author guidance for avoiding the upfront wrap.** When the extension can be expressed as a `Sequence` or `Collection` requirement rather than `Array`-specific, library authors should prefer the wider protocol — it lets users pay leaf-injection cost lazily per element rather than upfront for the whole array:

```swift
// Preferred when feasible:
extension Sequence where Element == Int | String {
    func summary() -> Int { reduce(0) { $0 + 1 } }
}

let xs: [Int] = [1, 2, 3]
xs.lazy.map { e -> Int | String in e }.summary()
// O(1) memory peak; the leaf-injection wrap fires per element during iteration,
// total work O(N) but no upfront array allocation.

// Falls back to upfront wrap when the extension must be on Array:
extension Array where Element == Int | String { func summary() -> Int { self.count } }
(xs as [Int | String]).summary()
// O(N) upfront wrap + 4× transient memory peak; fix-it suggested by the compiler.
```

The two forms are not equivalent — the `Sequence` form gives up index-based access on the receiver and any `Array`-specific API. Authors targeting random-access workloads or array-shape methods (`reversed()`, slicing) need the `Array` form and pay the upfront wrap. Authors targeting iteration / map / reduce workloads should prefer `Sequence` / `Collection` for the lazier cost profile. The implicit-lift future direction would close this gap by emitting specialised bodies that pay nothing for the wrap when the body doesn't actually need narrowed-`Any` element layout (see [Per-element leaf injection at the extension boundary](#per-element-leaf-injection-at-the-extension-boundary)).

### Conformance synthesis (v1 scope)

Three classes of protocol conformance for `A | B`, in order of how they reach the runtime witness:

| Class                          | Examples                                                                                                  | What v1 does                                                                                                                                                                                                                                                                                                                                      |
| ------------------------------ | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Self-conforming**      | `Error`, marker protocols (`Sendable`, `AnyObject`, `Copyable`)                                   | The existential layout itself acts as the witness — same `SelfProtocolConformance` Swift already uses for `any Error`. **Fully working in the prototype.** No new compiler code required.                                                                                                                                              |
| **Untagged structural**  | `Codable`                                                                                               | Compiler synthesises encode/decode that walks the alternation in declaration order: encode emits the leaf value directly with no wrapper or discriminator; decode tries each leaf in order until one succeeds.**Working in the prototype** (see § [Implementation status](#implementation-status)). § [Issue 5](#issue-5-codable-round-trips) describes the design and decode-error model. |
| **Per-witness dispatch** | `Hashable`, `Equatable`, `Comparable`, `CustomStringConvertible`, others with method requirements | Requires real witness-table emission with thunks that open the existential and dispatch to each leaf's own conformance.**Deferred from v1** — see [Future directions § Per-narrowed-Any witness emission](#per-narrowed-any-witness-emission). The v1-era escape hatch for these protocols is explicit `as! any P`:                        |

```swift
let v: Int | String = 42
let data = try JSONEncoder().encode(v as! any Encodable)  // works today
```

Once the per-witness synthesis lands in a follow-up, the `as!` becomes redundant and the user code collapses through three stages, each with the standard Swift diagnostic chain:

```swift
encode(v as! any Encodable)   // ⚠ "Forced cast … always succeeds; use `as`"   (fix-it: drop `!`)
encode(v as any Encodable)    // OK — explicit existential erasure, allowed by convention
encode(v)                     // OK — implicit erasure, same path Swift takes for `encode(5)`
```

So existing v1-era `as!` adoption code automatically rots into actionable warnings as the compiler grows the conformance — the migration is mechanical. The same chain applies in let-binding and `return` positions; assignment-style `let p: any P = v` is the standard `T: P → any P` erasure rule.

**Why route 1 (per-witness dispatch) is the design we commit to**, even though v1 ships without the synthesis: the alternative — auto-erasing `A | B` into `any P` whenever every leaf conforms to `P` — would lose [Spelling is identity](#spelling-is-identity) at the protocol-dispatch level, because two values with different spellings would observably behave the same once erased. v1 leaves the synthesis door open so the conformance can land later without changing the user-visible model.

**User-defined extensions on `A | B` are deferred to a follow-up.** v1 ships only the synthesised conformances above; user-written `extension Int | String { func ... }` and `extension Int | String: P { ... }` are rejected with a tailored diagnostic that points users at the v1 workarounds (extend each leaf type individually, or write a generic function with `where T: A | B`). When the follow-up lands, `Int | String` is treated as a first-class extension target — methods and conformances declared on it directly **take priority over the synthesised behaviour** above, the way a concrete type's own witness already shadows a protocol-extension default in Swift today. So `extension Int | String: Codable { ... }` for libraries with bespoke wire formats (OpenAPI discriminator, custom try-order beyond declaration order) is the user-override hook for the v1 untagged-Codable synthesis, not a re-declaration of an existing conformance: the v1 synthesis is an *implicit fallback*, not a *declared* conformance, so Swift's "no duplicate conformance" rule does not fire. See [Future directions § Extending a narrowed-`Any` directly](#extending-a-narrowed-any-directly) and [§ Codable user override](#codable-user-override). The v1 rejection is *not* a permanent design choice; it is the conservative starting point that defers the mangling work to a focused follow-up.

#### Cast safety

The cast surface (`as?` / `as!` / `is` / `case let _ as T:`) gets two correctness layers on top of `swift_dynamicCast`. Statically, the compiler walks both source and target for `NarrowedAnyType` and applies the rules in [Cross-shape conversion](#cross-shape-conversion): disjoint = hard error in every form; same spelling = redundant-cast warning; subset with different spelling = "always succeeds; use `as`" warning on `as?` / `as!`. Dynamically, the runtime's `swift_dynamicCast` path is layered with a leaf-membership post-check that re-applies the closed-leaf-set predicate the `Any`-singleton layout would otherwise lose, so nested `(Int | String) | Bool` casts compare against concrete-leaf metadata rather than the nested narrowed-`Any`'s `Any`-singleton.

#### Retroactive conformances and exhaustiveness

A subtle case [raised on the forums by ksluder][ksluder-post-55]: if a downstream module imports a third-party type and *retroactively* makes it conform to a protocol that one of the alternatives in `A | B` already conforms to, does that change the joined interface? And does adding a retroactive conformance silently turn a previously exhaustive switch into a non-exhaustive one?

**The join.** A retroactive conformance can only *add* protocols to the join, never remove them. The join of `A | B` is the *intersection* of `{P : A: P}` and `{P : B: P}`; both sets monotonically grow as new conformances arrive at link time. The visible interface of `Int | String` therefore widens, never narrows, when a downstream module retroactively conforms `Int` and `String` to a new protocol. This is identical to how `any P & Q`'s join behaves under retroactive conformance today, and we inherit that behaviour without ceremony.

**Exhaustiveness.** Switch over `A | B` is exhaustive over the *declared* alternatives, not over conformers of the join. A retroactive conformance that happens to make `String` conform to a protocol someone else's switch case mentions cannot turn a previously exhaustive `switch` into a non-exhaustive one — the alternatives are spelled by name, fixed at the point `A | B` is written, and remain exhaustive against that closed set. The retroactive conformance only affects which *additional* methods are dispatchable through the join.

### Generics: `where T: A | B`

A constraint `where T: A | B` reads as **set membership**: `T` is a member of the closed leaf set `{A, B}`. The constraint is *order-free* — `where T: A | B` and `where T: B | A` accept the same set of substitutions, because what matters is whether `T`'s dynamic type lies in the leaf set, not how the constraint was spelled.

```swift
func process<T: NetworkError | DecodingError>(_ error: T) {
    switch error {
    case let e as NetworkError:  ...   // exhaustive: the constraint guarantees T is one of the two
    case let e as DecodingError: ...
    }
}

// All three call sites are accepted by the constraint:
process(NetworkError.timeout)         // leaf passed; T = NetworkError under set-membership
process(DecodingError.malformed)      // leaf passed; T = DecodingError under set-membership
let e: NetworkError | DecodingError = ...
process(e)                            // alternation passed; T = NetworkError | DecodingError
```

The body type-checks against the *join* of the constraint's leaves — the body of `process` may use any member that every leaf provides, but cannot use leaf-only members without first narrowing with `as?`. This matches what the body of a function taking `A | B` directly already sees. (v1 implements this constraint via the same-type degraded form `T == A | B`, so the body always sees `T` as the alternation regardless of what the caller passed; full per-leaf binding is the [True set-membership](#true-set-membership-for-where-t-a--b) future direction. The set-membership reading at the *constraint* level is independent of the body-binding axis.)

**One narrowed-`Any` constraint per type parameter.** Multiple narrowed-`Any` clauses (`where T: A | B, T: C | D`) and the three `&`-with-narrowed-`Any` interactions (`(A | B) & P`, `(A | B) & SomeClass`, `(A | B) & (C | D)`) are **all rejected** with a diagnostic. Reasons and fix-it details are spelled out in § [Issue 6](#issue-6-generic-constraints-where-t-a--b) and § [Issue 9](#issue-9-interaction-with--protocol-composition--superclass).

**Throws position.** `throws(A | B)` is a *type position* (not a constraint position): the spelling is part of the function signature's identity. A protocol method declared `throws(NetworkError | DecodingError)` and an implementation written `throws(DecodingError | NetworkError)` are two *different* signatures; the compiler reports "does not conform to protocol" with a fix-it that reorders the implementation. Constraint-position order-freeness does not extend to type-position. § [Issue 6](#issue-6-generic-constraints-where-t-a--b) covers the asymmetry.

<a id="try-propagation-is-per-leaf-not-per-spelling"></a>
**Try-propagation is per-leaf, not per-spelling.** Spelling-as-identity at the throws *signature* (above) does *not* extend to the `try f()` propagation site. Propagation is a value-flow check: each leaf the inner call could throw must have a home in the enclosing function's declared throws set, regardless of how either side spelled the alternation. Concretely:

```swift
func doSomething()    throws(NetworkError | FilesystemError) { … }
func doAnotherThing() throws(FilesystemError | NetworkError) { … }   // same leaves, different spelling
func doMore()         throws(NetworkError) { … }                     // strict subset of leaves

func doBothThings() throws(NetworkError | FilesystemError) {
    try doSomething()       // leaves {NetworkError, FilesystemError} ⊆ outer set ✓
    try doAnotherThing()    // leaves {FilesystemError, NetworkError} ⊆ outer set ✓ — spelling differs, no cast needed
    try doMore()            // leaves {NetworkError} ⊆ outer set ✓ — leaf-injection at the throws boundary
}
```

The runtime thrown value is always *one concrete leaf* — both leaves of `doAnotherThing`'s declared set are accepted by `doBothThings`'s declared set, so the propagation type-checks without cost. This is the same posture [SE-0413] already takes for `throws(SpecificError) → throws(any Error)` widening, generalised from "subtype of `any Error`" to "leaf-set subset of the outer's declared set with per-leaf class/protocol subtyping". Spelling-as-identity still applies at *type-identity* boundaries — value-level cross-shape assignment, protocol-witness conformance, mangling — but those are signature questions, not propagation questions. The composition story in [Motivation](#motivation) (`O(N+M)` rather than `O(N×M)`) leans entirely on this rule: cross-library throws sets compose at the propagation boundary, not at the signature boundary.

<a id="uninhabited-never-leaves-and-the-inhabited-subset-rule"></a>
**Uninhabited (`Never`) leaves and the inhabited-subset rule.** `Never` is the bottom type — no value of type `Never` can be constructed. When `Never` appears as a leaf in a narrowed-`Any`, the runtime can never see a value of that leaf, so call-site **reachability** checks ("can this throw?", "is this case reachable?", "does this cast have any chance of succeeding?") are decided against the *inhabited* subset of the leaf set, not the static leaf set. Concretely:

- `throws(A | Never)`: inhabited subset `{A}` → `try` required at call site, exactly like `throws(A)`.
- `throws(Never | A)`: same inhabited subset after stripping the `Never` leaf; same call-site behaviour.
- `throws(Never | Never)`: inhabited subset empty → non-throwing at the call site, the natural multi-leaf extension of [SE-0413]'s existing `throws(Never)` rule. Calling such a function does not require `try`.
- `switch v` over `v: A | Never`: no `case _ as Never:` arm required for exhaustiveness — that leaf is unreachable, so omitting it is not a "missing case" diagnostic.
- `Codable` for `A | Never`: encode never sees a `Never` value (the dynamic type can never be uninhabited, so the type-directed dispatcher never reaches that arm); decode skips the `Never` leaf in the declaration-order try sequence (constructing a `Never` value would always fail). Operationally identical to `A`'s `Codable`.

**Type identity is *not* affected.** `A | Never` and `A` have different mangled names, different signatures, different witness-table identities — [Spelling is identity](#spelling-is-identity) is preserved at the ABI / Codable declaration order / witness-selection level. The inhabited-subset rule applies only to *call-site reachability decisions*, not to type identity itself. This is the same posture SE-0413 already takes for `throws(Never)`: the function's static type still mentions `Never`, but the call-site `try` requirement is computed from "is the throws set effectively empty?".

This rule answers @Nobody1707's `throws(Err<Never, Never>)` worry from the [pitch thread](https://forums.swift.org/t/pitch-narrowed-any/86369/12) — the enum desugar can't apply this rule because the case constructors are first-class declarations the type-checker has nowhere to hang the collapse on; narrowed-`Any`'s leaf set is visible to the type-checker at every use site, so the rule is mechanical to extend.

**Open question for review** (Pitch-stage TODO, not a fixed v1 commitment): should the inhabited-subset rule extend to extension `where Element == ...` matching? Currently *no* — spelling-as-identity wins, so an extension declared on `Array where Element == Int | Never` does *not* apply to `[Int]` values even though leaf sets are operationally equivalent. Extending Never-collapse to extension lookup would re-introduce an exception that subverts spelling-as-identity. The recommended v1 stance keeps extension matching strict-spelling and routes the user to the [Order-insensitive marker in `where` clauses](#order-insensitive-marker-in-where-clauses) future direction if they explicitly opt in.

**v1 conservatism + two future relaxations.** What v1 *does* give you at constraint position is leaf-set order-freeness: `where T: A | B` and `where T: B | A` are the same constraint. What v1 *does not* give you is (a) full set-membership specialisation — the body sees `T` as the alternation, not as the bound leaf when bound — and (b) cross-spelling matching at extension-`where Element == ...` clauses, where v1 honours [Spelling is identity](#spelling-is-identity). Both relaxations are sketched as Future directions: § [True set-membership for `where T: A | B`](#true-set-membership-for-where-t-a--b) addresses the binding axis, and § [Order-insensitive marker in `where` clauses](#order-insensitive-marker-in-where-clauses) addresses the spelling axis. They are independent and share the same constraint-solver hook (per-binding sorted-leaves identity).

### Exhaustiveness diagnostics

A `switch` over a narrowed-`Any` value is exhaustive iff every leaf is matched by at least one arm. The exhaustiveness checker reports missing leaves by name:

```swift
let v: Int | String | Bool = ...
switch v {
case let n as Int:    ...
case let s as String: ...
}   // error: switch must be exhaustive
    //   missing case: 'as Bool'
    //   fix-it: insert 'case let _ as Bool:' before the closing brace
```

Adding a leaf to a published `throws(A | B)` declaration (`throws(A | B | C)`) emits the same warning at every existing `do { try … } catch` whose arms used to cover `A | B` exhaustively. The warning is *not* silent; if the user has not opted into `@unknown default`-style fallback (narrowed-`Any` does not currently support `@unknown default` — the closed set makes it unnecessary), every consumer of the API sees the gap on the next compile.

### Mangling

A new mangling operator `XN` extends the existing `X`-namespaced operator subgrammar. The subscript-style EBNF:

```
narrowed-any-mangling ::= 'XN' decimal type-list ; the integer is leaf count
type-list             ::= type+
```

So `Int | String` mangles as `XN2SiSS` (count 2, then `Int`, then `String`); `(Int | Double) | String` as `XN2XN2SiSdSS` (outer count 2, first leaf is itself a narrowed-`Any` of count 2, then `Int`, `Double`, then the outer's second leaf `String`). Spelling-as-identity is preserved in the mangling: `Int | String` and `String | Int` mangle to different symbols (`XN2SiSS` vs `XN2SSSi`), so they have distinct ABI identities even when their leaf sets are equal.

Lower-case `n` in the `X`-namespace was already taken by parameter packs (`Xn`); upper-case `N` was free. No collisions with existing manglings.

### Grammar (EBNF)

```
type ::= simple-type
       | optional-type                         ; existing: T?
       | type '?'                              ; existing
       | function-type                         ; existing: T -> U
       | tuple-type                            ; existing: (T, U)
       | array-type                            ; existing: [T]
       | dictionary-type                       ; existing: [K: V]
       | existential-type                      ; existing: any P
       | metatype-type                         ; existing: T.Type / T.Protocol
       | narrowed-any-type                     ; new

narrowed-any-type ::= type ('|' type)+

/* `|` is a *type-level* operator at this position; in expression
    contexts it remains the bitwise-or operator unchanged. The
    type-vs-expression decision happens at the parser level by the same
    contextual-typing path that distinguishes `Int?` (optional type)
    from `?` (postfix optional in expression).                          */
```

Two sharp edges to keep in mind:

* `where T: A | B` and `throws(A | B)` parse `A | B` as a *constraint* term, not a type term — the spelling overlaps but the interpretation is set membership at the constraint level (see [Generics](#generics-where-t-a--b)). At the type-position level (function signature, including `throws(...)`), the spelling is part of the signature's identity.
* `(A | B)` is unambiguous in type position; in expression position it is a parenthesised bitwise-or expression. The disambiguation is contextual, not syntactic.

## Source compatibility

`A | B` is a brand-new spelling: in a type position (after `:`, `->`, `as`, `is`, inside `<…>`, inside `throws(…)`, inside `[…]` when the bracket opens an array or dictionary type, and after the `any` keyword) `|` is a type-list separator; everywhere else it remains the existing `BitwiseOr` operator. The disambiguation rides on the same parser context bit that already distinguishes `Int?` (optional type) from postfix `?` on an expression. **No existing valid Swift parses differently under this proposal**: `(T).self | (U).self` is still a bitwise-or on metatype expressions, `a | b` in expression position is still `BitwiseOr.|(a, b)`, and there is no token reservation that retires identifiers.

The only type-checking interaction worth noting is overload resolution. An `A | B` typed argument injects implicitly into a parameter typed `A | B` (same shape) and into any supertype overload of the join, but it does *not* implicitly convert to `B | A` (different shape — the cast must be written). Cross-shape conversion needs explicit `as`, by design (see [Spelling is identity](#spelling-is-identity)). This rule is conservative: a method typed `(Int | String) -> Void` is unaffected by the introduction of a sibling `(String | Int) -> Void`, even when both are visible.

## ABI compatibility

`A | B` lowers to the same existential layout as `Any` (24-byte inline value buffer + 1-word type-metadata pointer; value-witness pointers reach through the metadata). Adding the feature is **purely additive**: no existing type's metadata kind, layout, mangling, or witness-table convention changes. The new `XN` mangling operator extends the existing `X`-namespaced operator subgrammar without colliding (lower-case `n` is taken by Pack; upper-case `N` was free). A standard library that does not declare any narrowed-`Any` typed entity emits no narrowed-`Any` symbols and is bit-identical to a build without this proposal.

For library authors who *do* expose a narrowed-`Any` in their public API, the alternative-list itself is part of the type identity and therefore part of the ABI. Adding or removing an alternative changes the mangled name and is an ABI break, exactly as adding or removing an enum case is. The break direction depends on the position the alternation appears in:

| Position                                                         | Adding an alternative                                                                                       | Removing an alternative                                                                              |
| ---------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| **Function return** `() -> A \| B`                        | Source-additive — callers' switches lose exhaustiveness, but values still flow. ABI break.                 | Source-break — old callers may have written `case let _ as B: …` that now never fires (warning). |
| **Function parameter** `(_ x: A \| B) -> ()`              | Source-break — callers used to pass leaves of `A \| B`, now must inject through the wider set. ABI break. | Source-additive — old callers pass narrower set, fine.                                              |
| **Stored property** `var x: A \| B`                       | Source-break in both directions — getter and setter both visible.                                          | Same.                                                                                                |
| **Typed throws** `throws(A \| B)`                         | Source-additive at the*throw* site, source-break at the *catch* site. ABI break.                        | Source-break at throw site.                                                                          |
| **Generic constraint** `where T: A \| B` (set membership) | Strictly additive — adding leaves the existing constraints satisfied.                                      | Source-break — uses that relied on the removed leaf no longer satisfy the constraint.               |
| **Internal-only**                                          | Free — no ABI surface to break.                                                                            | Free.                                                                                                |

The pattern matches enum case evolution. For library authors who want to grow an already-published alternation list across an ABI boundary without breaking clients, the recommended patterns are the same ones that exist for enums today: gate the new spelling behind `@available(…)` so old clients see the old type and new clients see the new one, or wrap in an enum from day one and grow the enum under `@unknown default`.

(The `@available` here is the standard *library-evolution* pattern — the same one Swift libraries already use to add an enum case, an API overload, or a new public method without breaking older clients. It is *not* a gate on whether `A | B` itself can be compiled on a given target — that question is answered separately in [Implications on adoption](#implications-on-adoption), and the answer is "no gate needed, the feature back-deploys".)

## Implications on adoption

The narrowed-`Any` runtime story is purely additive and does not require a new runtime version. **No OS upgrade, no minimum-deployment-target bump, and no `@available` annotation are needed to adopt the feature** — `A | B` back-deploys to any Swift-supporting OS the user already targets. (The §[ABI compatibility](#abi-compatibility) section above mentions `@available` in a *different* context: it is the standard library-evolution pattern for *growing an already-published alternation list*, just like adding an enum case in v2 of a library. That pattern applies once you have shipped a public API; it has nothing to do with whether you can use `A | B` in your code today.)

Concretely:

- Every narrowed-`Any` value reuses the **`Any` singleton metadata pointer** at runtime — `Any`'s metadata has been in the Swift runtime since 1.0.
- Every cast goes through the existing **`swift_dynamicCast`** entry point — same runtime symbol used today by `cat as? Animal` and `v as? any P`.
- The closed-class-of-conformers table (point #6 of the [compile-time check](#compile-time-check)) is **emitted by IRGen as static linked data in the user's binary**, not as a new runtime artefact; the runtime simply reads it via the existing conformance-descriptor lookup path.
- The new conformance kind for synthesised protocol conformances (the per-witness dispatch route, [deferred from v1](#per-narrowed-any-witness-emission)) is a compile-time concept; the witness tables it eventually produces will be normal witness tables full of SIL thunk functions, dispatched via existing `witness_method` machinery.

In other words: every "new" thing this proposal introduces lives either in the *type-checker* (two-layer cache, leaf-set classification, conformance synthesis) or in the *user's binary* (closed-class-of-conformers tables, witness thunks). The runtime side has nothing new to learn. Code compiled against an older runtime continues to run correctly when the same source is re-compiled with a newer toolchain that supports `A | B` — the same back-deployment story Swift has for SE-0335 `any P`, property wrappers, result builders, and similar compile-time-only features.

Library adopters should be aware of three points. First, exposing a narrowed-`Any` in public API binds the library to that alternative list — adding or removing an alternative is a separate decision from the binary-compatibility question above. Second, the synthesised `Codable` conformance is *untagged*; libraries that round-trip narrowed-`Any` through their own wire format may want to wait for the user-extension override (see [Future directions § Codable user override](#codable-user-override)) to ship a custom encoder/decoder pair, or hand-roll an enum wrapper for the field in question. Third, a narrowed-`Any` cannot be exposed through `@objc` selectors, because Objective-C has no spelling for the closed type set; selectors that need to cross the bridge should expose `Any` instead, paired with a Swift-side narrowing helper.

The feature can be freely adopted and un-adopted in source code without affecting source compatibility, with one exception: switching from a narrowed-`Any` typed property to a wrapper enum (or vice versa) is a stored-property change and is therefore a source break in both directions. Package authors selectively adopting depending on tools-version availability should use the upcoming-feature-flag mechanism (the proposal will register an `Upcoming Feature Flag` name once the flag is allocated).

## Open syntax issues and proposed resolutions

Each issue states the question, the resolution chosen, and a one-paragraph rationale. Issues whose resolution is the more contentious half of the design are marked ⚠.

### Issue 1: Parsing `|` in type vs. expression context

**Question.** `|` is `BitwiseOr` in expression context today. Reusing it for type-list separation needs to be unambiguous.

**Resolution.** Parser already has the type/expression context bit (the same one that distinguishes `Int?` from postfix `?`); reuse it. In a type position, `|` is the type-list separator; in an expression position, it remains `BitwiseOr.|`. Concretely: after `:`, `->`, `as`, `is`, inside `<…>`, inside `throws(…)`, after `any`, inside `[…]` when the bracket opens an array/dictionary type — those are the type-position triggers.

### Issue 2: Function-type parenthesization

**Question.** `A | B -> C` is ambiguous between `A | (B -> C)` and `(A | B) -> C`.

**Resolution.** Parens are required *only* in positions where the alternation would otherwise be ambiguous with another type-level construct. In any position with a `:` label closing the type context (e.g. `(_ value: Int | String) -> Void`, `(_: Int | String, Int)`), parens are *not* required. Anonymous parameter / return / tuple-element positions still require parens when the alternation contains a `->` or comma that could rebind. § [Source-level surface](#source-level-surface) lists the exact form.

### Issue 3: Interaction with `Optional<T>` and `T?`

**Question.** `T?` is sugar for `Optional<T>`. Does `T | nil` work? Does `T | T?` collapse?

**Resolution.** `nil` is a literal, not a type — `T | nil` is a *parser error* with a fix-it suggesting `T?` (which is unrelated to the alternation). `T | T?` is a legal narrowed-`Any` with leaves `{T, Optional<T>}` — no collapse, no warning, per [Spelling is identity](#spelling-is-identity). The two leaves are different runtime types (`T` vs `Optional<T>` carry different metadata) so the cast machinery works as expected.

### Issue 4: Overlapping types

**Question.** `T | T`, `Cat | Animal` (where `Cat: Animal`), `Int | (Int | String)`, etc. — should the compiler warn or normalise?

**Resolution.** Neither. The compiler is silent on these cases, treats them as written, and lets the pattern-match exhaustiveness checker do the heavy lifting. The user wrote a specific spelling because they wanted that spelling. The only structural diagnostic that fires is the size warning (§ [Issue 7](#issue-7-large-or-deeply-nested-narrowed-any)), and that is purely about cognitive load.

### Issue 5: Codable round-trips

**Question.** How does `A | B` encode and decode?

**Resolution.** Untagged. Encoding writes the leaf value directly with no wrapper or discriminator. Decoding tries each leaf in declaration order until one succeeds. If all leaves fail, the prototype currently propagates the *last* leaf's underlying decode error verbatim; the proposal commits to wrapping all-fail in a fresh `DecodingError.typeMismatch` annotated with the full leaf set as a final-polish item (see § [Implementation status](#implementation-status)).

```swift
let v: Int | String = 5
JSONEncoder().encode(v)                         // produces `5`
typealias V = Int | String
JSONDecoder().decode(V.self, from: jsonData)    // succeeds on either an integer or a string at that key
```

For overlapping pairs (`URL | String`, `Int | Double`) declaration order is the user's controlled lever — `Int | Double` decodes a JSON number as `Int` first, `Double | Int` decodes the same number as `Double`. § [Spelling is identity](#spelling-is-identity) is what makes the order well-defined.

<a id="codable-debugger-interaction"></a>
**Debugger interaction with the try-each-leaf strategy.** The synthesised `init(from:)` invokes each leaf's own `init(from:)` via SIL `try_apply`. When a leaf's decoder body throws (e.g. `String.init(from:)` finding a JSON number where it expected a string), the throw goes through Swift's runtime `swift_willThrow` hook *before* the synthesised body silently catches it and tries the next leaf. **Xcode's "Swift Error Breakpoint" is keyed on `swift_willThrow`**, so it fires once per failing leaf attempt during a multi-leaf decode — even when the overall decode ultimately succeeds.

This is not specific to narrowed-`Any` — any `try?`-based fallback chain (`if let n = try? c.decode(Int.self) { … }; if let s = try? c.decode(String.self) { … }`) and the manual wrapper-enum pattern (the canonical untagged-Codable workaround today) have the same behaviour. The synthesised body's debugger interaction is the same as the manual idiom it replaces. Foundation's own `JSONDecoder` triggers identical breakpoint behaviour for several internal try-each paths (heterogeneous `oneOf`-style schemas, `decodeIfPresent`-with-type-mismatch, etc.).

The root cause is the `Decoder` protocol's deliberate omission of any "peek" / "is-this-decodable-as-T" API — Codable is encoder/decoder-agnostic by design, and the protocol exposes only `decode(_:)` (which throws) plus `decodeNil` / `contains` / `count` (which can't tell type). Some wire formats (JSON, plist) could in principle expose wire-level type tags before decoding; others (custom binary, CSV) physically cannot without schema. The protocol picks the lowest common denominator so user code stays format-portable; the operational consequence is that any try-each-leaf fallback chain — synthesised or hand-written — must invoke each leaf's actual decoder, which throws.

**Compiler-side mitigations** (in increasing order of scope), each with its own trade-off:

1. **SIL-level `[suppress_will_throw]` flag on `try_apply`.** A new SIL attribute that the synthesised `init(from:)` attaches to each leaf's `try_apply`. The runtime's throwing path checks the caller's flag (via TLS or per-frame state) and skips the `swift_willThrow` invocation when set. Targeted: only the synthesised speculative attempts are silenced; user code's throw breakpoint remains intact. **Cost:** runtime ABI surface change (new flag, runtime cooperation), and debug-info loss for the silenced attempts (users who *want* to see "where exactly did it fail" lose that signal even on the leaf path that ultimately succeeds).

2. **Optimizer-driven inline + throw-elimination** for cases where the leaf's `init(from:)` is `@inlinable`. SILOptimizer inlines the body, observes that the only consumer of the thrown error is `destroy_value` in the synthesised body's `errBB`, and rewrites the throw site into a non-throwing tagged-result return path. `swift_willThrow` is never called because there is no longer a `throw` instruction at the leaf site. **Cost:** only works when the leaf's `init(from:)` is in scope for inlining (same module or `@inlinable`); Foundation's `JSONDecoder.SingleValueContainer.decode(_:)` is *not* `@inlinable`, so the common-case decoders don't benefit. Best treated as an opportunistic optimisation, not a guaranteed fix.

3. **Format-specific fast-path in the synthesised body.** Compiler synth detects when the inner decoder is a "peekable" stdlib type (`JSONDecoder`'s internal container, `PropertyListDecoder`'s, etc.), and dispatches directly on the wire type (`JSONValue` tag in `JSONDecoder`'s case). Falls back to try-each for unknown decoders. **Cost:** brittle compile-time coupling between the synth and specific stdlib decoder implementations — every new format-specific decoder needs synth-side support; private internal types of `JSONDecoder` (`_JSONDecoder` etc.) aren't part of Codable's public ABI, so the compiler reaching into them is layering-violating. Also doesn't solve the problem for third-party decoders.

4. **Add a `peek` API to the `Decoder` protocol.** A new `peekedType: WireType?` property on `Decoder` (or `func peek<T>(_:) -> Bool`), with a default implementation returning `nil`. The synth uses peek when available; falls back to try-each when `nil`. JSONDecoder, PropertyListDecoder, and other tree-based formats override; binary/streaming formats that physically can't peek leave it at `nil` (and therefore still trigger `swift_willThrow` on speculative attempts, but the failure mode at least has a documented public name now). **Cost:** Codable protocol surface change — a separate SE proposal in its own right; would need a `WireType` taxonomy ABI-stable across decoder kinds; coordination with the Foundation team to land the override.

The proposal commits to **#1 as the v1+1 follow-up** (most contained scope, biggest immediate UX win for Codable-heavy users) and lists #2–#4 as longer-term avenues. v1 ships the try-each behaviour with the breakpoint UX as-is, matching every other `try?`-based fallback chain in Swift today.

The decoder path is implemented in the prototype, with untagged round-trip across nested narrowed-`Any`, `Codable` containers, and arrays of narrowed-`Any` (see § [Implementation status](#implementation-status) for the full prototype matrix). The v1 untagged synthesis acts as a **fallback** that fires when no user-declared extension provides Codable on the narrowed-`Any` (see [§ Extending a narrowed-`Any` directly](#ext-rule-fallback) for the priority rule). **User-extension override** — `extension Int | String: Codable { ... }` for libraries that want a custom encoder/decoder pair (tagged on the wire, OpenAPI discriminator, custom try-order beyond declaration order) — is deferred to a follow-up; v1 rejects all extension forms on a narrowed-`Any` target, so in v1 the synthesis fallback is the only path. See [Future directions § Codable user override](#codable-user-override) for the design and the implementation gating (extending the mangler / Sema to accept narrowed-`Any` as an extension target).

### Issue 6: Generic constraints `where T: A | B`

**Question.** What does `where T: A | B` mean? Does order matter? Can you write multi-clause `where T: A | B, T: C | D`? Can `&` appear anywhere?

**Resolution.** *Set membership.* `where T: A | B` means `T`'s dynamic type lies in the closed leaf set `{A, B}`. The constraint is **order-free** — `where T: A | B` and `where T: B | A` accept the same set of substitutions. Once `T` is bound to a concrete leaf, the substitution recovers spelling-as-identity at every type-position use of `T`.

**Single narrowed-`Any` constraint per type parameter.** Multi-clause (`where T: A | B, T: C | D`) is rejected — there is no obvious correct meaning (intersection? union of unions?) and the diagnostics for the two interpretations are confusing. § [Issue 9](#issue-9-interaction-with--protocol-composition--superclass) covers the related ban on `&`-with-narrowed-`Any`.

**Constraint position is order-free; type position is not.** `throws(A | B)` is a *type position* (the spelling is part of the function signature's identity); a protocol method declared `throws(A | B)` and an implementation written `throws(B | A)` are two different signatures, and the compiler reports "does not conform to protocol" with a fix-it that reorders the implementation. The constraint-position order-freeness from `where T: A | B` does *not* leak out to type position.

### Issue 7: Large or deeply-nested narrowed `Any`

**Question.** Should `Int | UInt | Int8 | UInt8 | Int16 | UInt16 | Int32 | UInt32 | Int64 | UInt64 | Float | Double` (twelve leaves) be diagnosed?

**Resolution.** Soft-warn at **8 flat leaves** (`narrowed_any_size_warning`), suggesting "consider using a named enum or a generic constraint with a sealed protocol". Threshold chosen empirically from a sweep of stdlib + Swift test suites: 95th percentile of in-use enum case counts is 8.

The warning is suppressible per use site; it is not an error. There is no warning for nesting depth as such — `(A | B) | C` (depth 2, 3 leaves) is fine; `(A | B | C | D | E | F | G | H | I)` (depth 1, 9 leaves) hits the threshold.

### Issue 8: Tuple of narrowed-`Any` vs. narrowed-`Any` of tuples

**Question.** Is `(Int | String, Bool)` the same as `(Int, Bool) | (String, Bool)`?

**Resolution.** No. They are different types with different mangled names. `(Int | String, Bool)` is a 2-tuple whose first element is a narrowed-`Any`; the runtime carries a 2-tuple value with the first slot containing an `Any`-singleton metadata pointer. `(Int, Bool) | (String, Bool)` is a narrowed-`Any` whose leaves are themselves tuples; the value is a single `Any`-singleton with a tuple metadata pointer.

The two have different `as` rules (the first allows independent leaf injection per element; the second requires casting between full tuple types). Pattern matching destructures them differently as well. This is a special case of [Spelling is identity](#spelling-is-identity).

### Issue 9: Interaction with `&` (protocol composition / superclass)

**Question.** What does `(A | B) & P` (mixing narrowed-`Any` with protocol composition) mean?

**Resolution.** Rejected. All three forms are diagnosed as errors:

- `(A | B) & P` — a narrowed-`Any` intersected with a protocol composition: ambiguous (does it filter the leaves down to those conforming to `P`? add `P` to the join?).
- `(A | B) & SomeClass` — a narrowed-`Any` intersected with a class: same ambiguity.
- `(A | B) & (C | D)` — two narrowed-`Any`s intersected: the leaf-set intersection question is decidable but breaks the [depth-1](#the-depth-1-principle-structurally-nested-behaviourally-flat) principle.

In all three, the diagnostic computes the candidate intersection and offers a fix-it that writes the intersected narrowed-`Any` directly (e.g. `(Int | String) & Comparable` → fix-it: write `Int | String` and rely on the join, or narrow the constraint to a generic with `where T: Int | String, T: Comparable` if that's what was meant). Protocol composition `&` between *protocols only* (no narrowed-`Any` on either side) remains legal as a leaf of a narrowed-`Any` — `(P & Q) | (R & S)` is fine.

## Implementation status

Prototype on a fork of `swiftlang/swift` at [miku1958/swift][fork-swift], branch [`narrowed-any/phase1-poc`][fork-branch]. 12 lit tests at [`swift/test/NarrowedAny/`][fork-tests], 12/12 pass in ~17 seconds (11 runtime-positive `%target-run-simple-swift` files exercising end-to-end behaviour + 1 `%target-typecheck-verify-swift` file locking in the negative-path Sema diagnostics). The bullets below list every user-visible capability the prototype exercises today; the gap list afterward names what still has to land before the proposal is reviewable as a v1.

Working today:

- Parser, `NarrowedAnyType` AST node, mangling (`XN` operator), `.swiftmodule` / `.swiftinterface` round-trip.
- Cross-shape `as` / `as?` / `as!` runtime via `swift_dynamicCast` with closed-leaf-set post-check.
- Pattern matching exhaustiveness, `switch` over narrowed-`Any` with leaf-naming arms.
- Self-conforming protocols (`Error`, `Sendable`, marker protocols) — full per-leaf dispatch via existential layout.
- Untagged `Codable` round-trip across nested narrowed-`Any`, `Codable` containers, arrays of narrowed-`Any` (Issue 5 design).
- `typed throws` end-to-end — exhaustive cross-domain `catch`, async / await transparent, rethrow-scope leak fixed.
- [Per-leaf try-propagation](#try-propagation-is-per-leaf-not-per-spelling) — `try f()` accepts cross-spelling and leaf-subset propagation modulo Never (leaf-set subset wins, spelling-as-identity stays at the function-signature boundary). The runtime path is a SIL-level unchecked cast on the inner thrown value (Any-singleton layout is identical across spellings, so bytes don't move).
- [Inhabited-subset rule for `Never` leaves](#uninhabited-never-leaves-and-the-inhabited-subset-rule) — `throws(A | Never)` requires `try` like `throws(A)`; `throws(Never | Never)` is non-throwing at the call site (multi-leaf extension of [SE-0413]'s `throws(Never)` rule); `switch v: Int | Never` is exhaustive without a `case _ as Never:` arm; the leaf-aware "missing case" hint never suggests `_ as Never`. Type identity is unchanged.
- Generic `where T: A | B` (currently lowered to same-type degraded form `where T == A | B` — see [Future directions § True set-membership](#true-set-membership-for-where-t-a--b) for the full rule).
- Set / Dict / Array stdlib integration, KeyPath, reflection.
- Constraint solver tuple-leaf injection, enum-case dispatch, cross-domain catch arms.
- `-O` SILOptimizer integration.

Known v1 gaps (must land before review or planned for first follow-up):

- **Per-element leaf injection fix-it at the extension boundary**: when a leaf-typed receiver (`xs: [Int]`) reaches an extension on a narrowed-`Any` element (`extension Array where Element == Int | String`), v1 emits the existing same-type-requirement error with a fix-it that inserts `(receiver as [Int | String])` before the call. The fix-it makes the cast one keystroke; users see "error → click → fixed". Wired in `lib/Sema/CSDiagnostics.cpp`'s `RequirementFailure::diagnoseAsError` — fires when the requirement is `SameType`, the rhs is `NarrowedAnyType`, and the lhs is a leaf of the rhs (recursively, including nested narrowed-`Any`); locks in by `diagnostics.swift §8b` verify-mode annotations. The *implicit* form (no cast at all) is deferred to a follow-up — see [Future directions § Per-element leaf injection at the extension boundary](#per-element-leaf-injection-at-the-extension-boundary). Explicit-cast (`(xs as [Int | String]).method()`) is verified working end-to-end (test bed `phase2_edge.swift §8`); the SIL path for the cast itself is mature.

  **Cross-spelling diagnostic-quality gap** (`zs: [Int | String]` reaching `extension Array where Element == String | Int`): the diagnostic for that axis surfaces through a *different* code path — per-alternative type comparison emitting "any Int" vs. "any String" — which doesn't carry narrowed-`Any` context, so the fix-it above doesn't activate. The cross-spelling cast itself works end-to-end via explicit `(zs as [String | Int]).method()` (runtime-free relabel, validated in `phase2_edge.swift §8`); only the diagnostic quality is below the leaf-injection bar. A separate Sema follow-up reshapes the diagnostic emission to recognise the narrowed-`Any` parent types and attach an analogous fix-it. Marked as a known v1 limitation in `diagnostics.swift §8a`; not blocked on v1 review.
- **Per-witness dispatch** (`Hashable`, `Equatable`, `Comparable`, `CustomStringConvertible`, etc.): synth path today is gated on `isMarkerProtocol() || requiresSelfConformanceWitnessTable()`. v1 escape hatch: explicit `as! any P` (works, `phase2f-runtime.swift §8a` validates). See [Future directions § Per-narrowed-Any witness emission](#per-narrowed-any-witness-emission) for the design.
- **swift-syntax sync**: companion fork at [miku1958/swift-syntax][fork-syntax], branch [`narrowed-any/syntax-sync`][fork-syntax-branch] (commit [`2973425f`][fork-syntax-commit]). Adds 3 syntax nodes — mechanical schema change + parser loop — to teach `swift-syntax` to recognise `A | B` natively. The branch is published but not yet upstreamed; the ABI / API surface is small and review-ready, but it ships in lockstep with the language change so the upstream PR will land alongside the swift-evolution proposal acceptance.

The prototype's test bed (`swift/test/NarrowedAny/`) exercises 12 lit tests covering the capabilities above. Cross-module compilation verifies the alternation round-trips through `.swiftmodule` and `.swiftinterface` formats; `-O` regression verifies optimisation-level transparency; the verify-mode `diagnostics.swift` locks in the negative-path Sema diagnostics (disjoint cast errors, cross-spelling extension dispatch, the `extension Int | String { … }` non-nominal note) so future Sema work can't silently regress them.

## Future directions

Items deliberately deferred from v1. Each is a follow-up proposal in its own right; v1 leaves the design door open without committing to a specific surface.

### Per-narrowed-Any witness emission

The conformance synthesis for protocols with method requirements (`Hashable`, `Equatable`, `Comparable`, `CustomStringConvertible`, and most user-written protocols) is the largest deferred item. v1 ships with the explicit `as! any P` escape hatch (see [Conformance synthesis (v1 scope)](#conformance-synthesis-v1-scope)); the follow-up replaces that with compiler-emitted witness tables containing SIL thunks that open the existential, cast into each leaf in declaration order, and forward via `witness_method` against the leaf's own conformance.

Why a separate proposal: the SIL verifier rejects the obvious shortcut (short-circuiting `lookupExistentialConformance` to `getSelfConformance(P)`), so this needs a genuinely new `BuiltinProtocolConformance` flavour (`NarrowedAnyDispatch`) plus IRGen work to emit the thunks, plus linker-coalesce-friendly mangling so duplicate `(A | B, P)` thunks across translation units collapse. Each piece is mechanical but adds Sema and IRGen surface that benefits from a focused review.

Once it lands, v1-era `as! any P` adoption code automatically rots into actionable warnings — the migration is mechanical.

### Witness-merge: replacing multiple overloads with one narrowed-`Any` signature

> *Throwing this out for community design work, not proposing it.*

A protocol with multiple overloads of the same nominal method **that share a return type** but differ in parameter type and/or thrown error could be implemented with a single narrowed-`Any` method whose signature unions the parameter and throws axes:

```swift
protocol Encoder {
    func encode(_ value: Int)    throws(NetworkError)  -> Data
    func encode(_ value: String) throws(DecodingError) -> Data
    func encode(_ value: User)   throws(AuthError)     -> Data
}

// Today the implementer writes three overloads. Under witness-merge, one merged form would suffice:
struct G: Encoder {
    func encode(_ value: Int | String | User) throws(NetworkError | DecodingError | AuthError) -> Data
}
```

This works because **parameter type and `throws` are both axes the caller deals with one leaf at a time**, while the return type is what the caller depends on to do further work:

- *Parameter*: each call site passes exactly one leaf value. Widening the parameter from `A` to `A | B | C` is contravariant on the caller side — the caller's existing call-site values still fit, the wider union just lets the implementer accept them all in one body.
- *`throws`*: each call's `try` either succeeds or throws *one* leaf error. Widening the throws from `E1` to `E1 | E2 | E3` doesn't break existing `catch` clauses — the caller's handlers still match the leaves they handled before, the new leaves get covered by additional arms (or `default` in untyped catch).
- *Return type*: this is what makes merge **inadmissible** when the overloads differ on it. If the protocol declared three overloads with returns `Int`, `String`, `User` and the implementer merged them into a `→ Int | String | User`, every call site that previously typed `let n: Int = encode(...)` against a specific overload now has to accept the union back and runtime-dispatch on the leaf — the caller's static contract has been silently weakened from "I know I will get an Int" to "I will get one of three things and need to narrow". That is the same trapdoor [Spelling is identity](#spelling-is-identity) protects against in the value-type direction, applied at the witness boundary.

So the rule the future-direction design has to commit to is: **merge is only available when every overload in the merged set has the same return type**. Param / throws axes are unionable; return axis must already be uniform.

Non-trivial design questions left for the follow-up proposal even within that constraint:

- **Sub-protocols.** If `Encoder2: Encoder` adds a fourth overload, does the implementer of `Encoder2` have to re-merge the wider set? Does the inherited merged form cover the new overload by leaf-set extension, or does that count as silent widening of the contract?
- **Super-classes.** A class implementing one overload and a subclass overriding a different overload — under merge, the super-class's merged signature must commute with each sub-class's specialisation. The contravariance rules on the param axis across the inheritance edge are not obvious; current Swift override rules require exact-type-match per overload.
- **Overload resolution at the call site.** If a type provides both the overload set *and* a merged narrowed-`Any` form, how does the type-checker pick? Today's overload resolution is built around exact-type-match plus implicit conversions; "implicit narrowed-`Any` injection at the param" would blur which overload wins. The merged form being strictly wider on parameter helps — the leaf-specific overload is always at least as specific — but a clean rule still has to be written.
- **`throws` interplay with [SE-0413].** Typed throws under SE-0413 already lets the implementer narrow the protocol's declared throws set to a subset; the merge form would be the dual move (widening across overloads). The two should compose cleanly but the precise rule depends on what witness-boundary variance the variance-relaxation future direction also commits to (see § [Variance at the protocol-witness boundary](#variance-at-the-protocol-witness-boundary)).

v1 ships with the conservative rule — each overload needs a separately-named implementation, no merge — and lets experience with witness-merge accumulate before any specific surface is proposed.

### Variance at the protocol-witness boundary

v1 commits to *exact-spelling match* at the protocol-witness boundary: a protocol method declared `(A | B) -> C` and a conformer's implementation differing in parameter / return / throws spelling does not satisfy the requirement. The conservative rule lets the type-checker reuse the existing exact-match witness lookup.

A future relaxation could permit the implementation to use a wider parameter type or narrower return type (i.e. function-type subtype variance applied to narrowed-`Any` leaf sets):

```swift
// Hypothetical relaxed rule — not in v1.
protocol P {
    func f(_ x: Int | String) -> Int | String | Bool
}
struct S: P {
    func f(_ x: Int | String | Bool) -> Int { ... }   // wider param, narrower return
}
```

The relaxation is strictly more permissive than v1's rule, so adopting it later does not break existing code. It does require Sema to compute leaf-set inclusion at the witness boundary and IRGen to emit a parameter-narrowing / return-widening thunk. Same shape as throws variance under [SE-0413]'s chosen rule.

### Per-element leaf injection at the extension boundary

v1 ships extension dispatch on `Array where Element == A | B` as **explicit-cast-with-fix-it**: a leaf-typed receiver (`xs: [Int]` reaching `extension Array where Element == Int | String`) is rejected with a fix-it that inserts `(receiver as [Int | String])`. The cast walks the array element-wise, wrapping each leaf as an `Any`-singleton existential, because `[Int]` (8-byte stride) and `[Int | String]` (32-byte stride: 24-byte buffer + 8-byte metadata) have different per-element layouts — the conversion cannot be a runtime-free relabel.

A follow-up could relax the cast to be **implicit**, but the valuable target is *not* "remove the keyword from sight" — it's **specialisation**: when `xs.summary()` is called with `xs: [Int]`, the compiler emits a body specialised on `Element = Int` rather than wrapping the receiver upfront. The body's `self.count`, `self.isEmpty`, `for e in self` all work on the leaf-typed receiver with **zero per-element wrap**; only individual statements that genuinely require narrowed-`Any` layout (`let v: Int | String = self[i]`, passing `self` to another function expecting `Array<Int | String>`) trigger a single-element or whole-buffer wrap at that point. For methods that don't need narrowed-`Any` element layout — by far the common case for "give my heterogeneous-array some helper methods" — this means **zero runtime cost** instead of v1's O(N) upfront wrap.

This is a substantially harder follow-up than the v1 fix-it suggests. The two implementation paths split:

- **Path A — Whole-receiver coercion (the cheap version of "implicit").** Sema, at the leaf-injection extension-dispatch site, binds `Element` to the narrowed-`Any` and inserts an implicit `CoerceExpr` wrapping the receiver — equivalent to rewriting `xs.summary()` to `(xs as [Int | String]).summary()` before SILGen sees it. SILGen already lowers the explicit `as` cast correctly. Runtime cost is unchanged from the explicit form: O(N) wrap once per call site. **This buys nothing at runtime over v1's fix-it; it just removes the cast keyword.** The CSApply work is real (a few days) but the user-visible win is small.

- **Path B — Per-element-axis specialisation (the valuable version).** SILOptimizer or a dedicated specialisation pass emits a per-leaf-element variant of the extension method. The body type-checks against `Element = Int | String` (so `switch self[i]` over leaf cases still type-checks), but the SIL is generated with `Element = Int`, with leaf-injection wraps inserted only at the points the body actually demands narrowed-`Any` layout. For methods that never demand the wider element type (count, isEmpty, indexing, iterating without leaf-typing), specialisation produces zero-overhead leaf-element bodies. This is the win that justifies the future direction.

Path A is mechanical; Path B requires a new SILOptimizer pass that understands narrowed-`Any` leaf injection at the element axis (analogous to how the existing generic-specialiser handles `T : SomeProtocol`). Both share the constraint-solver detection step (recognising the leaf-injection situation), but only Path B avoids the O(N) runtime cost. The v1 fix-it is the stop-gap; Path A is the next-step ergonomic win; Path B is the destination.

### Order-insensitive marker in `where` clauses

Spelling-is-identity makes `[Int | String]` and `[String | Int]` distinct types, so generic code parameterised on `[T]` matches one shape only and every other shape needs an explicit cross-shape `as` at the call site. A future surface could introduce a where-clause marker — strawman `where T == Int | String unordered` or `where T ~= Int | String` — that opts a single generic binding into matching against any spelling whose sorted leaves agree, so inside such a generic both `[Int | String]` and `[String | Int]` satisfy `[T]` directly. The constraint-solver hook parasitises the [two-layer cache](#compile-time-check): comparing the binding's interned sorted-leaves identity instead of its canonical-type pointer is `O(1)` after first encounter, no new caches required. Spelling-is-identity stays the global default; this is opt-in for code that genuinely doesn't care (set-algebra over leaves, JSON/Codable wrappers that round-trip through a sorted-canonical wire format, the IDE-side cross-shape completion below sharing the same index). Two open design questions: marker spelling needs evolution review, and protocol-witness dispatch needs an explicit decision on whether marker-bound dispatch follows the caller's spelling (preserves the dispatch-level "spelling is identity" guarantee that v1's [conformance-synthesis escape hatch](#conformance-synthesis-v1-scope) inherits) or normalises to sorted-canonical form (cheaper but loosens that guarantee — the same loosening v1 was framed to avoid).

### SourceKit completion auto-inserts cross-shape cast

Today the user writing `arr.xxx()` where `arr: [Int | String]` and `xxx()` is declared on `[String | Int]` gets nothing in the completion list. SourceKit could surface those methods anyway, by indexing extensions per *sorted-leaves identity* (free as a side-product of the [two-layer cache](#compile-time-check)) and rewriting the picked completion to wrap the receiver in the right cast: `(arr as [String | Int]).xxx()` for total, `(arr as? [Int])?.xxx()` for partial. Same shape as Swift's existing completion-time enrichment (`try` / `await` insertion, `@dynamicMemberLookup` resolution): the language rule still rejects implicit cross-shape, but the IDE writes the visible cast for you. Tooling, not language.

### Macros consuming narrowed-`Any` syntax

A macro that *generates* `A | B` works today by textual expansion. A macro that wants to *receive* a narrowed-`Any` and reflect over its alternatives needs `swift-syntax` to expose a `NarrowedAnyTypeRepr` accessor. The underlying syntax node already exists in the prototype's swift-syntax fork ([miku1958/swift-syntax][fork-syntax], branch [`narrowed-any/syntax-sync`][fork-syntax-branch]); reflecting it through `MacroExpansion` is a follow-up macro-proposal.

### Result builders: simplifying `buildEither` and the `_ConditionalContent` ladder

Result builders ([SE-0289](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0289-result-builders.md)) introduced `buildEither(first:)` / `buildEither(second:)` for `if / else` branches whose two arms produce different `Component` types. Today every result builder author has to invent a sum-type wrapper — SwiftUI's [`_ConditionalContent<T, F>`](https://developer.apple.com/documentation/swiftui/viewbuilder/buildeither(first:)) is the canonical example — and an `if / else if / else` chain produces a *nested* wrapper tree (`_ConditionalContent<_ConditionalContent<A, B>, C>`) whose depth scales with branch count.

`A | B` collapses both pieces of this ceremony:

```swift
// Today (SwiftUI's ViewBuilder, abridged):
@resultBuilder struct ViewBuilder {
    static func buildEither<T: View, F: View>(first:  T) -> _ConditionalContent<T, F>
    static func buildEither<T: View, F: View>(second: F) -> _ConditionalContent<T, F>
}

// With narrowed-`Any` — wrapper struct retired:
@resultBuilder struct ViewBuilder {
    static func buildEither<T: View, F: View>(first:  T) -> T | F
    static func buildEither<T: View, F: View>(second: F) -> T | F
}
```

Two follow-on improvements ride on the same change:

- **Flat `n`-ary `if / else if / else`.** A three-arm chain can produce a flat `A | B | C` rather than the depth-2 `_ConditionalContent<_ConditionalContent<A, B>, C>` ladder. Whether to emit flat or nested is a deliberate result-builder design choice — flat preserves user-source order and is what most readers expect; the nested form is what existing builders happen to produce because it's all `_ConditionalContent` could express.
- **Loop bodies with heterogeneous arms.** `buildArray` over a `for` whose body branches between `A` and `B` could produce `[A | B]` directly instead of `[_ConditionalContent<A, B>]`. Same erasure improvements apply.

Open design questions for the follow-up proposal:

- **Spelling-as-identity at builder boundaries.** `if cond { A() } else { B() }` produces `A | B`; `if !cond { B() } else { A() }` would produce `B | A`. Per [Spelling is identity](#spelling-is-identity) those are different types — different mangled names, different Codable order, different `View` witness selection. Today's `_ConditionalContent<T, F>` already has this property (the type parameter order encodes the source order); narrowed-`Any` makes it more visible. The right design probably commits to *source order* as the rule and surfaces a "consider reordering branches if Codable order matters" warning at large builders.
- **Same-type collapse.** `if cond { A() } else { A() }` produces `A | A` per spelling-as-identity (two leaves of the same type, no idempotence). Today `_ConditionalContent<A, A>` is similar — it works, with no perf benefit over `A` alone. Should the result builder's `buildEither` collapse same-type branches to `A`? If yes, that's a builder-side rule, not a language rule (the language preserves `A | A` per Issue 4). The cleanest answer is "leave the spelling alone; if the user wants collapse they can write the branches without the `if`".
- **`_ConditionalContent` source compatibility.** SwiftUI ships `_ConditionalContent` as part of its public ABI for backwards compatibility. Migrating SwiftUI itself to narrowed-`Any` would be an ABI-additive change (SwiftUI's `View` body is `some View`, so the concrete type is opaque to callers); whether SwiftUI does the migration is a separate library-level decision, but it is *unblocked* by narrowed-`Any` shipping.
- **Pre-narrowed-`Any` builders.** Existing builders using `_ConditionalContent`-style wrappers continue to work unchanged — narrowed-`Any` doesn't deprecate them. New builders or new versions can adopt narrowed-`Any` whenever they want.

The language change is already done by v1; this is purely a library-evolution follow-up, and a particularly visible one because SwiftUI is one of Swift's most-used result builders. v1 leaves this open as a library-side opportunity rather than committing the SwiftUI team to specific timing.

### Tagged-union layout for small POD leaf sets (and the Embedded Swift story)

When every leaf is small and POD (`Bool | UInt8`, `Int8 | Int16`, `Int32 | Float`), the existential layout is wasteful — a 24-byte inline buffer plus a 1-word metadata pointer (32 bytes total) for what could fit in two-to-eight bytes. An IRGen pass could lay out such narrowed-`Any` values as `discriminator + payload` locally, the way `Optional<Int>` packs into a spare-bit representation today. Cross-module ABI continues to use the existential layout; the optimisation is local-only.

**The qualifier list is narrow.** Leaf sets that **don't** qualify for this follow-up:

- **Any leaf that isn't POD.** Reference-counted types (`String`, `URL`, classes, `any AnyObject`), types containing reference counts (`String` because of its storage backing, `Optional<class>`, `Array<T>`), and any type whose value-witness table demands non-trivial copy/destroy. Tagging requires the runtime to never look at the value-witness table for storage management; non-POD leaves break that.
- **All-POD leaf sets without a spare bit the tag can borrow.** The optimisation packs `discriminator + payload` into a single value; the discriminator needs at least one bit to distinguish leaves. The only way to avoid an extra tag byte is to **borrow a spare bit pattern from one of the payloads** — and that requires the largest leaf to leave some bit pattern unused. Concretely:

  | Leaf set | payload | spare bit available? | unaligned size | aligned stride |
  |---|---|---|---|---|
  | `Bool \| UInt8` | 1 byte | ✅ `Bool` uses 1 bit, leaving 7 spare | 1 byte | **1 byte** |
  | `Int8 \| Int16` | 2 bytes | ✅ tag in `Int16`'s sign-bit-adjacent free region | 2 bytes | **2 bytes** |
  | `Int32 \| Float` | 4 bytes | ✅ tag in `Float`'s NaN payload (~2²² free patterns) | 4 bytes | **4 bytes** |
  | `Int \| Double` | 8 bytes | ✅ tag in `Double`'s NaN payload (~2⁵² free patterns; classic NaN-boxing) | 8 bytes | **8 bytes** |
  | `Int \| UInt32` | 8 bytes | ❌ `Int` uses the full 64-bit range; `UInt32`'s zero high bits collide with low `Int` values, so they can't be the tag | **9 bytes** | **16 bytes** |
  | `Int \| Int` (legal under spelling-as-identity) | 8 bytes | ❌ same reason | 9 bytes | 16 bytes |

  The "spare bit available" column is the load-bearing one. `Int | UInt32` is all-POD and small, but `Int` saturates its 64-bit range with no spare bit pattern, and `UInt32` stored in an 8-byte slot has its high 32 bits as zeros — which collide with low-positive-integer `Int` values (`UInt32(7)` and `Int(7)` are bit-identical when both stored in a 64-bit slot). With no spare bit to borrow, the tag needs a standalone byte; 8-byte payload + 1-byte tag = 9 bytes, which on `Int`'s 8-byte alignment rounds up to 16 bytes per element. Still beats 32 (Any-singleton), but doesn't hit the word-size sweet spot that `Bool | UInt8` or `Int | Double` reach.

So `Int | String`, `URL | Data`, `NetworkError | DecodingError` (where each Error case carries reference-counted message strings or wrapped types) — these stay on the existential layout. The follow-up's primary beneficiaries are typed-throws error sets where every leaf is a small POD enum (status codes, fixed-size error variants), and Embedded Swift workloads that can't tolerate `swift_dynamicCast` regardless of leaf shape.

**This layout is required, not just nice-to-have, for Embedded Swift.** Embedded targets restrict dynamic dispatch, existential boxing, and `swift_dynamicCast` — exactly the v1 mechanisms narrowed-`Any` reuses on full-Swift targets. Without the tagged-union layout, Embedded Swift cannot adopt the feature; with it, narrowed-`Any` lands in the same perf regime as a hand-written wrapper enum on Embedded (discriminator + max-payload size, no metadata indirection, no dynamic cast). [SE-0413]'s perf wins are concentrated in single-leaf typed throws (fixed-size stack-resident error, no boxing); the moment a function wraps a multi-case enum to compose multiple error sources, payload + discriminator is already the regime — narrowed-`Any` under tagged-union layout matches that regime, just without the named-wrapper-enum tax. This means:

- **This proposal (v1, full-Swift, no Embedded)** ships with the existential layout + `swift_dynamicCast`. The `O(N+M)` ergonomic improvement over wrapper enums is delivered immediately; the runtime cost vs a hand-written wrapper enum is the existential indirection (an extra metadata pointer + the `swift_dynamicCast` lookup at the catch arm).
- **A follow-up proposal (Embedded Swift unblocked)** ships the tagged-union layout for small POD leaf sets (typed-throws errors usually fit), with `discriminator + payload` local emission and no `swift_dynamicCast` on the catch path. Cross-module ABI continues to use the existential layout, so libraries compiled against v1 keep working when re-imported under the follow-up.

The layout transformation is a local IRGen pass — no ABI changes, no new metadata kinds, no language-rule changes. It can ship as a separate SE proposal once the Embedded-Swift narrowed-`Any` story is mature enough for review.

### Reflection over the closed leaf set

`Mirror(reflecting:)` currently surfaces only the dynamic leaf via the standard existential machinery. A future reflection surface could expose the static leaf list — useful for diagnostics tooling and for macros that want to inspect a narrowed-`Any` shape they were given.

### Extending a narrowed-`Any` directly

Today `extension Int | String { … }` is rejected with a tailored "non-nominal type" diagnostic that points users at two workarounds: extend each leaf type individually (`extension Int { … }; extension String { … }`), or add behaviour uniformly across leaves through a generic function with `where T: A | B`. This is the conservative v1 starting point; it is **not** a permanent design choice.

A follow-up lifts the restriction by **treating `Int | String` as a first-class extension target** — both for adding methods and for declaring protocol conformances, gated on the same compiler work (extending the mangler / Sema to accept narrowed-`Any` as an extension target):

```swift
// Form 1 — adding a method directly to the narrowed-Any.
extension Int | String {
    func describe() -> String {
        switch self {
        case let n as Int:    return "int(\(n))"
        case let s as String: return "str(\(s))"
        }
    }
}

// Form 2 — declaring a user-supplied protocol conformance, the canonical
// motivator from Issue 5 (see § Codable user override). Takes priority over
// the v1 untagged-Codable synthesis fallback.
extension Int | String: Codable {
    func encode(to encoder: Encoder) throws { ... }
    init(from decoder: Decoder) throws { ... }
}
```

Two design rules anchor the follow-up:

<a id="ext-rule-leaf-reach"></a>
**Receiver's static type owns its method-dispatch priority.** Method lookup on a narrowed-`Any` value (or a leaf value of a narrowed-`Any`) follows Swift's existing most-specific-wins rule, applied to the type lattice with the `A <: A | B` subtype edge from § [Subtyping lattice](#subtyping-lattice):

1. **Receiver typed `A` (a leaf).** Lookup walks `A`'s own extensions first; if none has the method, walks up to `A | B`'s extensions (implicit `Any`-singleton box at the receiver — O(1), operationally identical to `let v: Int | String = leaf; v.method()`); finally falls back to standard Swift lookup (protocol conformances etc.).
2. **Receiver typed `A | B`.** Lookup walks `A | B`'s own extensions first; if none has the method, falls back to the *join* (members provided by every leaf via protocol conformance — see § [The join](#the-join-what-members-are-visible)).

So when both an `extension Int { describe() }` and an `extension Int | String { describe() }` are in scope:

```swift
extension Int          { func describe() -> String { "Int" } }
extension Int | String { func describe() -> String { "Int | String" } }

let a: Int          = 7;   a.describe()  // "Int"           — Int's own extension wins
let b: Int | String = 7;   b.describe()  // "Int | String"  — Int | String's own extension wins
let c: String       = "x"; c.describe()  // "Int | String"  — String has no own describe(), reaches via subtype lift
```

This is exactly the rule Swift already applies to class-extension dispatch (`extension Animal { … }` vs. `extension Dog: Animal { … }` resolve by receiver static type), generalised to narrowed-`Any` because `A <: A | B` is just another edge in the subtype lattice. Users can force the wider extension on a leaf-typed receiver via explicit `(value as Int | String).describe()` when they need the wider behaviour.

The lift to narrowed-`Any` extension methods is the natural lift of value-level leaf injection (`let v: Int | String = 7`) to the method-dispatch axis: same box, same cost (O(1) per call), same dispatch shape — only the source-level ergonomics differ.

**Container-axis dispatch** (`xs: [Int]` reaching `extension Array where Element == Int | String`) follows the same design intent — most-specific-wins, with subtype-lift as fallback — but ships in v1 as **explicit-cast-with-fix-it** rather than implicit because the per-element layout differs (8-byte raw `Int` slots vs. 32-byte `Any`-singleton slots) and the conversion is O(N); see [Per-element leaf injection at the extension boundary](#per-element-leaf-injection-at-the-extension-boundary). The *value*-axis lift is O(1) (single box) and matches the value-binding axis directly, so no such retreat is needed there.

<a id="ext-rule-fallback"></a>
**User-declared extensions take priority over the v1 synthesis fallback.** The auto-synthesised conformances from § [Conformance synthesis (v1 scope)](#conformance-synthesis-v1-scope) (untagged Codable, marker / self-conforming protocols, eventual per-witness dispatch) act as *implicit fallback witnesses* that fire when no user-declared extension provides the conformance. A user-written `extension Int | String: Codable { ... }` is the explicit declaration; the synthesis steps aside. There is no "duplicate conformance" conflict because the v1 synthesis is a fallback rule, not a declared conformance — same shape as how a concrete type's own witness already shadows a protocol-extension default implementation in Swift today.

Form 1 is the general "method-adding" surface; the body can dispatch per-leaf with `switch self` whose exhaustiveness over the closed leaf set is statically verifiable (strictly more amenable to type-checking than the corresponding `extension any P { … }` for an open-existential, which is also rejected in current Swift). Form 2 is § [Codable user override](#codable-user-override) — the originally-motivating use case from [Issue 5](#issue-5-codable-round-trips), allowing libraries to ship custom Codable wire formats without a hand-rolled wrapper enum. The two forms parse and type-check through the same path; the only extra piece for Form 2 is wiring the user-declared conformance through `lookupConformance` so it shadows the synthesis fallback at conformance lookup time.

### Parameter packs collapsed into a narrowed-`Any`

`each T` (SE-0393) describes a *positional* pack; `A | B | C` describes an *unordered closed set*. The two are dual. A future surface could let a parameter pack collapse into a narrowed-`Any` (`Pack { each T }` → `T1 | T2 | … | Tn`), but pack expansion is an operation whereas `A | B` is an identity, so the design is non-trivial.

### Codable user override

Issue 5 in v1 ships untagged-only Codable as a *synthesis fallback*. A follow-up should provide a user-extension hook for libraries that want a custom encoder/decoder pair — typically because their wire format expects a discriminator (OpenAPI's `oneOf`, serde-style tagged union) or a specific overlap-handling order beyond declaration order. The intended syntactic form is the natural one — a Codable conformance written as an extension on the narrowed-`Any`, taking priority over the v1 synthesis fallback per the rule in [§ Extending a narrowed-`Any` directly](#ext-rule-fallback):

```swift
extension Int | String: Codable {
    func encode(to encoder: Encoder) throws { ... }
    init(from decoder: Decoder) throws { ... }
}
```

This is the canonical motivator for lifting the v1 restriction on user-defined conformances over narrowed-`Any` (see [Conformance synthesis (v1 scope)](#conformance-synthesis-v1-scope)). The implementation gating is one piece of work — extending the mangler / Sema to accept narrowed-`Any` as an extension target — that simultaneously unblocks the more general [user-written extensions on narrowed-`Any`](#extending-a-narrowed-any-directly). Until that lands, libraries needing a custom Codable wire format hand-roll an enum wrapper for the field in question.

### True set-membership for `where T: A | B`

v1 lowers `where T: A | B` to the same-type degraded form `where T == A | B` — the constraint is satisfiable by the alternation type itself, but the body cannot specialise `T` to a single leaf. A full set-membership rule would let the body see `T` as a leaf when the substitution actually is one (and as the alternation when it isn't), which requires disjunctive requirements at the constraint solver level. This is a substantial constraint-solver project on its own.

A natural pairing for that work is § [Order-insensitive marker in `where` clauses](#order-insensitive-marker-in-where-clauses), which addresses the *spelling* axis of the same constraint surface: full set-membership lets the body specialise `T` to a single leaf when bound; the order-insensitive marker lets the binding match against any spelling whose sorted leaves agree. The two are independent — either can land first — but they share the same constraint-solver hook (per-binding sorted-leaves identity) and would benefit from a single round of design review.

### SIL-level optimisation passes

Today a `switch` over a narrowed-`Any` lowers to a chain of `checked_cast_addr_br` calls into `swift_dynamicCast`. Because the alternative list is closed and known at compile time, the chain can be sunk into a `load_metadata` plus a metadata-pointer `switch_value` table — the same shape `enum` dispatch uses. Likewise `v as? Int` against `v: Int | String` can be reduced to a single metadata compare. These are local SIL transforms, not ABI changes; they sit in `lib/SILOptimizer` and apply on top of whatever route the conformance-synthesis design eventually lands on.

## Alternatives considered

### Anonymous enums

```swift
let x: enum { case .a(Int); case .b(String) } = .a(42)
```

Internally consistent — every enum is just an explicit closed class of conformers — but inverts the ergonomic problem: the user must invent a discriminator name (`.a`, `.b`) for every alternation, and pattern matching becomes a syntactic choice between `case .a(let n)` and `case let n as Int`. Two ways to express the same thing. Narrowed `Any` keeps one syntax (`as`), unifying with existing existential pattern matching.

### Generic `OneOf<A, B, ...>`

Possible today as a library type. Loses pattern-matching exhaustiveness, requires `.first` / `.second` discriminators per call site, and does not compose into typed throws (`OneOf3<A, B, C>` and `OneOf4<A, B, C, D>` are unrelated types). The community has tried this shape repeatedly and found the discriminator drift unacceptable in practice.

### Closed protocols / sealed (a hypothetical Swift feature)

> Note: `sealed protocol` is not currently a Swift feature. It has been discussed in the community (analogous to Kotlin's `sealed class`, Scala 3's `sealed trait`, Java 17's `sealed`) but an evolution proposal would be required to ship it.

A `sealed AppError: Error` would let you enumerate the conforming types, similar to closing a leaf set. But sealed-protocol-style proposals require declaring the protocol *and* declaring conformances at the protocol's site — they cannot anonymously enumerate "these three pre-existing types I need to compose". Narrowed-`Any` is the *anonymous* shape; a sealed protocol design would be the *named* shape, and the two are complementary.

### TypeScript-style structural unions

TypeScript's `string | number` is *structural*: a value-level set, members of the alternation have access to the structural intersection of properties (none, in that case), and the language's type erasure to JavaScript means dispatch is by `typeof` checks the user writes themselves. That model is incompatible with Swift's nominal typing and existential boxing — and prior Swift pitches that copied it have repeatedly failed review on exactly that point. This proposal is deliberately *unlike* TypeScript: members come from the join of nominal conformances, not from structural intersection; runtime dispatch is `is` / `as?` / `switch case _ as T:` against a closed conformer table, not user-written `typeof`; and there is no implicit cross-shape conversion.

### Comparison with how other languages spell the same idea

| Language | Syntax | Semantics |
| --- | --- | --- |
| Scala 3 | `A \| B` | Untagged sum; nominal leaves; **commutative** (`A \| B` ≡ `B \| A`); unboxed; member access via LUB. |
| Ceylon | `A\|B` | First-class type-level union; nominal leaves; `T?` desugars to `T \| Null`; commutative. |
| TypeScript | `A \| B` | Structural assignability over the alternatives; only the intersection of properties is visible; runtime narrowing by user-written `typeof` / `instanceof` checks. |
| Crystal | `Int32 \| String` | Compiler-inferred from the set of values a binding can hold; method must exist on every alternative. |
| Python (PEP 604) | `int \| str` | Type-hint sugar for `Union[int, str]`; no runtime semantics. |
| Kotlin | none — uses sealed classes | Nominal closed hierarchy. |
| F# | `type X = A of … \| B of …` | Discriminated union, *nominal*. |
| Rust | `enum X { A(A), B(B) }` | Discriminated union, *nominal*. |

This proposal sits **closest to Scala 3 and Ceylon in surface, closest to Swift's `any P` in semantics** — a nominal closed-class-of-conformers existential with the same boxing, opening, and `is` / `as` machinery `any P` already has, retargeted at a finite list of leaves rather than a single protocol's conformer set.

### Prior art: cast-feasibility algorithms

The previous subsection compared *spelling*. The compile-time question of "is this cast statically valid, statically partial, or statically impossible?" — what this proposal calls *cast feasibility* — is itself a well-trodden design point.

| Approach | Languages using it | What it does | Where this proposal sits |
| --- | --- | --- | --- |
| **Pattern usefulness** ([Maranget 2007][maranget]) | OCaml, Rust, Haskell, F# | Recursively walks patterns, computing which constructors are "useful" (covered / uncovered). For closed sum types this degenerates to "enumerate constructors and check coverage". Used for both exhaustiveness and reachability diagnostics. | This proposal's *switch exhaustiveness* check is structurally Maranget pattern usefulness applied to closed-conformer existentials — `case _ as A`, `case _ as B`, … patterns are checked for whether they collectively cover the declared leaf set. The *cast-feasibility* check (the subset / overlap / disjoint classification of two leaf sets at an `as` / `as?` / `as!` site) is simpler — sorted-leaf-set intersection with class/protocol subtyping at the leaf level — and shares the closed-conformer reasoning rather than the full pattern-vector algebra. |
| **Structural assignability with caching** | TypeScript | Recursive structural walk of `(source, target)`, memoised on the pair to handle cyclic types. The user-facing `x as T` is **trust-me** — no compile-time feasibility check beyond a lax "not totally disjoint" gate. | The two-layer cache borrows the memoisation idea but **rejects** the trust-me posture: disjoint is a hard error, subset-written-as-partial is a warning, every cast site is decided statically. |
| **Subtype lattice with `glb` / `lub`** | Scala 3 union types | Treats `A \| B` as a lattice element: `A \| B <: C` ⟺ `A <: C ∧ B <: C`; `C <: A \| B` ⟺ `C <: A ∨ C <: B`. Cast (`asInstanceOf`) is **runtime-only** on the JVM. | Per-leaf class/protocol subtype walks use the same lattice intuition, but feasibility is decided at compile time, not deferred to runtime. |
| **Subsumption (Roslyn)** | C# pattern matching | "Does this set of patterns subsume all reachable types?" Uses sealed-class info to bound the case set. Maranget-style with nominal subtyping. | Same subsume-the-closed-set posture, extended to *anonymous* closed sets created on the fly by `\|`. |
| **Trust-me cast** | Java `(Cat)x`, TypeScript `as`, Kotlin `as` | No compile-time feasibility check. Failure is a runtime `ClassCastException`. | Explicitly rejected. A closed leaf set makes "is this cast reachable" statically decidable, so the proposal decides it. |

Two takeaways for reviewers:

1. **Neither algorithm is novel.** Switch exhaustiveness over a closed leaf set is Maranget pattern usefulness specialised to closed-conformer existentials; cast feasibility is sorted-leaf-set intersection (an elementary set-relation classification) plus Swift's existing class-cast / conformance-lookup machinery applied per leaf. What is novel for Swift is choosing the *input shape* — an anonymous closed-class-of-conformers existential — and routing both checks through it.
2. **The strictness on disjoint casts is deliberate.** TypeScript and Java leave that decision to the runtime; Scala 3 erases the type entirely on JVM. This proposal goes the other way for the same reason `switch` exhaustiveness on `enum` is statically checked: the closed shape makes it decidable, so it should be decided.

## Acknowledgements

This proposal absorbs ideas from Wade Tregaskis, Michel Fortin, John McCall, Jordan Rose, Slava Pestov, Tino, and the Scala 3 / Ceylon design teams. The Codable design is a direct codification of the [2018 forum recommendation][itai-2018] by Itai Ferber (co-author of [SE-0166][SE-0166]). Disagreements are mine.

[SE-0166]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0166-swift-archival-serialization.md
[SE-0309]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md
[SE-0353]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0353-constrained-existential-types.md
[SE-0413]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md
[thread-67432]: https://forums.swift.org/t/structural-sum-types-used-to-be-anonymous-union-types/67432
[thread-70740]: https://forums.swift.org/t/se-0413-typed-throws-needs-union-types/70740
[thread-72574]: https://forums.swift.org/t/sum-types-type-disjunctions-and-possible-alternatives/72574
[thread-72668]: https://forums.swift.org/t/answered-disjunctions-in-types-why-is-this-something-that-the-type-system-cannot-and-should-not-support/72668
[thread-72700]: https://forums.swift.org/t/re-proposal-type-only-unions/72700
[fork-branch]: https://github.com/miku1958/swift/tree/narrowed-any/phase1-poc
[fork-swift]: https://github.com/miku1958/swift
[fork-syntax]: https://github.com/miku1958/swift-syntax
[fork-syntax-branch]: https://github.com/miku1958/swift-syntax/tree/narrowed-any/syntax-sync
[fork-syntax-commit]: https://github.com/miku1958/swift-syntax/commit/2973425f
[fork-tests]: https://github.com/miku1958/swift/tree/narrowed-any/phase1-poc/test/NarrowedAny
[itai-2018]: https://forums.swift.org/t/how-to-deal-with-completely-dynamic-json-responses/9441
[ksluder-post-55]: https://forums.swift.org/t/re-proposal-type-only-unions/72700/55
[maranget]: http://moscova.inria.fr/~maranget/papers/warn/warn.pdf
[miku1958]: https://github.com/miku1958
