---
title: Correct by Construction
tags: [architecture, types]
stage: budding
planted: 2026-07-03
tended: 2026-07-04
---

*Refined types, smart constructors, and where invariants come from.*

A general architecture guide — language-agnostic in spirit. Examples are
primarily in Python, and each major concept ends with compiler-verified
comparisons across **Rust, TypeScript, Go, and Haskell** (plus cameos from
Scala and F# where they offer something unique), because the concepts are
universal but what each language can *enforce* differs sharply.

The principle in one sentence: **design your representations so that invalid
things cannot be expressed at all — rather than expressed, then detected.**
"Things" is deliberately broad; the principle applies at (at least) four
scales:

- **Values** — only some strings are valid escaped pointers; only some ints
  are valid percentages.
- **States** — `{connected: bool, socket: Socket | None}` can express the
  nonsense combination `connected=True, socket=None`; a sum type
  (`Disconnected | Connected(socket)`) cannot — the bad combination has no
  encoding.
- **Sequences** — read-after-close type-checks when "open file" and "closed
  file" are the same type, and cannot when they aren't (typestate).
- **Relationships** — an index that is only meaningful for one particular
  collection; a date range whose start must precede its end.

This guide works the **values** scale in depth — it is the most common case
and the least language-dependent — and the ideas it develops (the leak
surface, recoverable vs. unrecoverable invariants, loud escape hatches,
runtime seams) recur at every scale. The other scales are queued in STATUS
as future sections.

At the values scale, the problem reads: you have a plain type (often a
primitive like `str` or `int`) where **only some values are valid**, and
validity comes from how the value was built — parsed, escaped, normalized,
range-checked. You want a type where holding a value *is* proof that the
building was done right.

There are two ways to try to get that proof, and the guide's thesis is that
they are not equally good:

- **Correct by validation:** accept any value, then check it — before use,
  at the door, wherever. The guarantee lives in the checks, and holds only
  where the checks actually ran.
- **Correct by construction:** make it impossible to *build* an invalid
  value. The guarantee lives in the type itself, and holds everywhere the
  value travels.

One term before the guide's sharpest idea. Call it a **leak** when an
invalid value ends up inside the type anyway — an unescaped string sitting
in a `SafeHtml`, wearing a name that promises it was escaped. The **leak
surface** — by analogy with "attack surface" — is the set of routes by which
that can happen.

The sharpest idea in the guide, worth reading twice — first in its general
form: **the leak surface equals what the public surface lets you express.**
What the "public surface" is depends on the scale: for states it is the
shape of the data (which field combinations have encodings at all); for
sequences it is which methods exist on which type. At this guide's scale —
values — the public surface is the constructor, so the maxim reads: **the
leak surface equals what the public constructor accepts.** If the public way
to make a `SafeHtml` accepts any string, then any string can become a
`SafeHtml`, and the name is a lie. If the only public way in takes raw text
and escapes it itself, then no unescaped value can exist. Everything else in
this guide — brands, smart constructors, escape hatches, runtime guards — is
refinement on that one axis: *what does the front door accept?*

Canon, if you want the sources (three angles on one idea): Alexis King's
["Parse, don't
validate"](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)
(validation should *produce a value in a better type*, not just say yes/no),
Yaron Minsky's "make illegal states unrepresentable" (design data so wrong
states have no encoding — chiefly the *states* scale above), and the Haskell
**smart constructor** idiom (hide the real constructor; export a function
that enforces the invariant).

---

## STATUS (working draft)

- [x] Running examples + JSON Pointer primer
- [x] Glossary
- [x] Point 1 — mixed-up same-typed values (1a invariant test, 1b vs kw-only,
      1c who produces + preventing bad wraps)
- [x] Point 2 — validity comes from how it's built (broken SafeHtml, Level
      A/B, full Pointer class, immutability, closure, payoff,
      total-vs-partial, naming convention)
- [x] Point 3 — The decision tree (two questions + two overlays; world-facts
      caveat, recoverable/unrecoverable definitions, identical-bytes litmus,
      direction-dependence, two hatch justifications, walkthrough table)
- [x] Recoverable vs unrecoverable invariants — absorbed into Point 3 (Q2
      definitions + SafeHtml litmus + Pointer direction-dependence)
- [x] Point 4 — Static vs runtime enforcement (annotations inert by
      themselves; four holes incl. Any-poisoning + mypy's documented
      Any-laundering exemptions (cast = named hatch, annotated assignment =
      silent hatch); parse-vs-guard split; subclass forgery + runtime
      finality; reconstruction bypass; one-guard-per-seam)
- [x] Point 5 — the capability matrix, rebuilt as the roll-up of the
      per-point cross-language sections (1d, Point 2, Point 4)
- [~] Point 6 — Proportionality / when to use which (recovered; wants
      deep-probe pass)
- [~] Point 7 — The provenance family + closing heuristic (recovered; wants
      deep-probe pass)
- [ ] NEXT SESSION — the "unrepresentable" rung: a level above Level A,
      where the invalid thing has no encoding (NonEmptyList as head+tail).
      Add it to Point 2's spectrum and as a Q0 before Point 3's Q2 ("can you
      pick a representation where the invalid thing can't be written
      down?"). Rework `Pointer` to store the token tuple and escape at
      render time (`__str__`) — making the escaping invariant
      unrepresentable rather than merely unreachable; keep the
      string-storing version as the contrast; re-verify all Pointer
      snippets + round-trip law, and update the walkthrough table.
- [ ] Illegal states unrepresentable — sum types; the *states* scale
      (seedling)
- [ ] Typestate: ordering by construction — the *sequences* scale (seedling)

Verified with mypy / runtime during design (snippets below are checked).
Points 5–7 recovered 2026-07-03 from the origin session (covjson-msgspec,
`d450b958`); their snippets were verified there. Point 4 probed and verified
here (Python 3.12, mypy 2.1, msgspec, pydantic 2.13, attrs 26.1).
2026-07-03: readability pass, then a second rework of Points 1–4 —
scenario-first headings, less assumed background, Python mechanics explained
inline; every claim in Points 1–2 re-verified locally (mypy 2.1 for the 1b
table; runtime for SafeHtml, `__slots__` mutability, the full `Pointer`
class, and the double-escape corruption chain).
2026-07-04: cross-language pass (sections 1d, "the same fortress," "the four
holes across languages," Point 5 matrix). Every Rust/Go/TypeScript/Haskell
snippet compiler-verified here — rustc 1.90, go 1.26, GHC 9.12,
TypeScript 6.0 `--strict` (+ bun for runtime) — including expected-failure
files proving each guarantee (Rust E0308/E0451; Go conversion and
unexported-field errors; TS `@ts-expect-error` under strict; GHC
type-mismatch and constructor-not-in-scope).

---

## Glossary (plain language)

- **Invariant** — a rule that is always true of every value of the type. For
  `Pointer`: "every `/` or `~` in a token's content was escaped." A type
  without an invariant has nothing to enforce; a type with one needs a way to
  enforce it.
- **Leak / leak surface** — a *leak* is an invalid value getting inside a
  type whose name promises validity (an unescaped string held by a
  `SafeHtml`). The *leak surface* — by analogy with "attack surface" — is
  the set of routes by which that can happen. In general it equals what the
  public surface lets you express; at this guide's values scale, what the
  public constructor accepts.
- **Constructor** — the code that makes a value. The **public constructor**
  is how outside code is *meant* to make one — and, per the leak-surface
  idea, the thing that decides how safe the type can be.
- **Brand** (also: newtype, nominal alias) — a type identical to its base at
  runtime but distinct to the type checker. It adds a **name, not a check**
  (Python's `NewType`).
- **Mint** — to produce a branded or refined value from a bare one
  (`UserId(row["id"])`), as a mint strikes coins: the act that turns raw
  material into certified currency. Like coinage, it is only legitimate when
  the right authority does it (Point 1c) — hence "concentrate the mints,"
  "re-mint at the boundary."
- **Refined type** — the general term for "a type whose values are a subset
  of a base type" (all the non-negative ints; all the escaped strings). A
  brand is a refined type with no enforcement; a value object is a refined
  type with enforcement.
- **Value object** — a small class that wraps a base value in order to
  enforce an invariant and give it real runtime identity.
- **Smart constructor** — a constructor that *enforces* the invariant;
  ideally the only sanctioned way to make the value.
- **Total vs. partial constructor** — a *total* constructor maps every input
  to some valid value (no failure path); a *partial* one can reject (raise).
  A "parser" is exactly a partial constructor.
- **Recoverable / unrecoverable invariant** — whether validity can be decided
  by examining a finished value alone. The load-bearing distinction of the
  whole guide; defined precisely in Point 3.
- **Seam** — any line where code with one set of guarantees meets code with
  another: type-checked interior against untyped caller, decoded objects
  against raw wire bytes, a specific type against an `Any`. Static
  guarantees end at seams; runtime guards belong exactly there (Point 4).
- **Static enforcement** — checks the type checker does before the program
  runs; sees only code it's asked to check.
- **Runtime enforcement** — checks actual code does while running
  (`if ...: raise`); sees everything, including callers the checker never
  saw.

---

## Running examples

Three examples recur, chosen because each sits at a different spot in the
design space:

- **A pair of ID types** (`UserId` / `ProductId`) — a type with *no
  invariant*: every `int` is a legitimate id value. The only possible bug is
  using one id where the other belongs. This is the home turf of
  roles/brands (Point 1).
- **JSON Pointer** — a type with an *unrecoverable* invariant (escaping).
  Once tokens are joined into a string, you can no longer tell whether
  escaping was done. Illustrates why some guarantees can only be made at
  construction (Points 2, 3).
- **`SafeHtml`** — a type whose invariant is real but which must sometimes
  accept a pre-made value it cannot rebuild. Illustrates the "loud,
  auditable escape hatch" (Point 2).

### The running example: JSON Pointer

A JSON Pointer ([RFC 6901](https://www.rfc-editor.org/rfc/rfc6901)) is a
string that names a location inside a JSON document — a sequence of
**reference tokens**, each introduced by `/`:

- `/axes/x` means "member `axes`, then member `x`"; `/values/0` means
  "member `values`, then array index `0`".
- The empty string `""` names the whole document.

Because `/` separates tokens, any `/` *inside* a token's content must be
escaped as `~1`, and any literal `~` as `~0`. So an object key `"x/y"`
becomes the token `x~1y`, and the pointer to it is `/axes/x~1y`.

(You'll also see a `#`-prefixed form, `#/axes/x~1y` — the same pointer
written as a URI fragment. We use the plain form here.)

That escaping rule — `/` becomes `~1`, `~` becomes `~0` — is the invariant
the examples care about. Full spec: RFC 6901 (terse; the primer above is
enough for this doc).

---

## Point 1 — Scenario: two kinds of values share one type and get mixed up

The situation. Your users have numeric ids. So do your products. Both are
plain `int`s, so this compiles, runs, and silently charges the wrong thing:

```python
def charge(user_id: int, product_id: int) -> None: ...

charge(product.id, user.id)     # swapped -- and nothing will ever complain
```

Every value involved is a perfectly good int. The bug isn't a *bad value*;
it's a *right value in the wrong place*. What you want is for the type
checker to treat "an int that identifies a user" and "an int that identifies
a product" as different things, even though the machine representation is
identical.

That's what a **brand** is for — in Python, `typing.NewType`. This point
covers when a brand is the right tool (1a), why the obvious cheaper
alternative doesn't cover the same ground (1b), and how to live with the
brand's one weakness (1c).

### 1a. First, ask: is there anything to check?

The question that decides between a brand and everything heavier in this
guide:

> Is every value of the base type a valid member of your type, or only some
> of them?

Put even more plainly: **is there such a thing as a "bad" `UserId`?** For
ids, no — any int could be someone's id. There is no rule a constructor
could enforce, nothing to check. When there is nothing to check, a pure
label is exactly the right amount of machinery, and `NewType` is that label:

```python
UserId    = NewType("UserId", int)
ProductId = NewType("ProductId", int)

def charge(user: UserId, product: ProductId) -> None: ...

u, p = UserId(5), ProductId(99)
charge(u, p)      # ok
charge(p, u)      # type error: arguments swapped  <- the only bug NewType prevents
```

The label costs nothing at runtime: `UserId(5)` *is* the int `5` (verified —
same object, same type). The distinction exists only in the type checker's
eyes, which is fine, because the mix-up is the only bug we were trying to
stop.

Now contrast with a type where "is this value valid?" is a *real* question.
Is `"a/b"` a valid JSON Pointer token string? That depends on whether the
`/` was supposed to be escaped — some strings are right and some are wrong.
Watch what happens if we reach for the same tool:

```python
Pointer = NewType("Pointer", str)   # WRONG TOOL: some strings are NOT valid
Pointer("a/b")                      #   pointers -- and this accepts any string,
                                    #   no questions asked
```

**The crux of Point 1:** the very same behavior — a brand's constructor
accepting any base value — is a **feature** for `UserId` and a
**catastrophe** for `Pointer`. For `UserId` there was nothing to check, so
checking nothing is correct. For `Pointer` there was something to check, and
the brand checked nothing. Same mechanism, opposite verdicts — so you cannot
decide "should this be a `NewType`?" by looking at code. You decide it by
answering the question at the top: *is there anything to check?* No → brand.
Yes → Point 2.

One honest caveat, so brands aren't oversold: even for `UserId`, a brand
won't stop someone who *deliberately* writes `UserId(some_product_int)`.
That's a cast — a conscious act. Brands prevent **accidents** (passing the
wrong variable), which is their entire job; no label stops lying.

Rule of thumb: **a brand distinguishes roles; it does not enforce
validity.** IDs, handles, opaque tokens, "which of these two same-typed
things is which" — all good brands, precisely because any base value is a
legitimate member.

### 1b. "Couldn't we just force keyword arguments instead?"

A reasonable objection: the swap bug in `charge(p, u)` could also be
prevented by making the parameters keyword-only, so every caller has to
write the names out:

```python
def charge_kw(*, user: UserId, product: ProductId) -> None: ...
def transfer(*, src: UserId, dst: UserId) -> None: ...       # same role twice
```

(The bare `*` in the signature means "everything after this must be passed
by name": `charge_kw(user=u, product=p)`, never `charge_kw(u, p)`.)

It's a fair idea, and it does catch some of the same bugs — but the two
tools protect different things:

> Keyword-only arguments are a property of one *function signature*,
> enforced at that function's call sites, and they rely on a human writing
> the right name next to the right value. A brand is a property of the
> *value itself*, enforced mechanically everywhere the value goes.

Here is what mypy actually catches in each case (each row verified,
mypy 2.1):

| Failure | kw-only alone | `NewType` |
| --- | --- | --- |
| Positional swap `charge(p, u)` | caught (forbids positional) | caught (type mismatch) |
| Right name, **wrong value** `charge_kw(user=p, product=u)` | MISSED | caught |
| Wrong role in **assignment** `uid: UserId = p` | N/A (not a call) | caught |
| Wrong role in **collection/return** `ids.append(p)` | N/A (not a call) | caught |
| Two args of the **same** role `transfer(src=b, dst=a)` | caught (names disambiguate) | MISSED (both `UserId`) |

Why keyword-only arguments alone don't cover it — two gaps:

1. **They stop *positional* swaps, not *wrong values*.** Forcing
   `charge_kw(user=..., product=...)` never checks that the thing after
   `user=` actually is a user id. The call `charge_kw(user=p, product=u)` —
   right names, swapped values — sails straight through and is caught only
   by the brand. In effect, kw-only converts "swapped by position" into
   "must be named," then trusts the human to name correctly. The brand
   removes the trust.
2. **They only exist at function calls.** Assignments, return values, dict
   keys, list elements — kw-only has nothing to say about any of them. A
   brand travels *with the value*, so a role mistake is caught anywhere the
   value goes.

And the case where the brand fails and kw-only is the only tool: **two
arguments of the same role.** In `transfer(src=..., dst=...)` both are
`UserId`s. The brand literally cannot tell them apart — swapping them
type-checks clean (verified: no error). Only the names stand between you and
sending money the wrong way.

So they compose rather than compete:

- Two *different*-typed values → often use **both**: the brand so the value
  can't be misused anywhere, kw-only for readability at the call.
- Two *same*-typed arguments → kw-only is your only option.
- A value that flows through assignments, returns, or collections → you need
  the brand; kw-only can't reach there.

### 1c. "What stops someone from wrapping the wrong value?"

The scenario. A teammate is deep in interior code, holding a bare `int`,
and the checker says "expected `UserId`." The shortest path to a green build
is obvious: `UserId(that_int)`. Red squiggle gone. What just happened — and
what stops it from happening wrongly?

Start by splitting the worry in two, using the 1a question:

- **If there's no invariant** (the correct brand use): then there is *no
  such thing* as a bad value to wrap — every int is a valid `UserId`. The
  only possible misuse is wrapping an int that plays a *different role*
  (`UserId(a_product_id)`): a **mislabel**. And since nothing distinguishes
  the two ints mechanically, no tool can catch it. Mislabels are a people
  problem, and the rest of this section is the people answer.
- **If there is an invariant:** a brand cannot prevent a bad wrap, period —
  and rather than fighting that, treat it as **the signal that a brand is
  the wrong tool**. Wanting to reject bad values at the wrap *is the
  definition* of needing a smart constructor (Point 2).

**Dissecting the "wrapped it to silence the checker" move.** When it
happens, there are exactly two possibilities, and only a human can tell
which:

- **(a) The value really is the right role — it just never got branded.**
  The proper fix is to brand it **upstream, at its source**, not at the
  error site. Wrapping where the error appeared "works," but now minting is
  scattered through the interior, where nobody can vouch for anything.
- **(b) The value is the wrong role.** The checker just caught a real bug,
  and the wrap **buries it**. This is the dangerous case.

Since telling (a) from (b) takes judgment, the design goal is to make raw
wraps **rare and conspicuous**: rare enough that each one gets looked at,
conspicuous enough that they can be found.

**Who should wrap, then? Branding is a boundary/authority job.** Think of a
brand as a **vouching**: writing `UserId(x)` asserts "I certify this int is
a user id." This guide calls that act **minting** — as in striking coins: it
turns a bare value into certified currency, and like coinage, it is only
legitimate when the right authority does it. The right minter is whoever can
actually make the assertion honestly — the layer where raw data first
becomes meaningful:

```python
# boundary (the outer layer that talks to the world -- DB, HTTP, files):
#   the AUTHORITY mints
def load_user(row: dict) -> User:
    return User(id=UserId(row["id"]), ...)   # the DB is the authority for user ids
def user_id_param(raw: str) -> UserId:
    return UserId(int(raw))                   # the request-parse boundary vouches

# interior (the inner logic that only computes): only CONSUMES
def greet(uid: UserId) -> str: ...            # takes a UserId; never writes UserId(...)
```

The boundary adapters — parsers, repositories, decoders — are authorities.
When the repository wraps `row["id"]`, it isn't checking anything (there's
nothing checkable about role; the DB simply *knows* these are user ids) —
it is asserting where the value came from, from a position to know. Interior
code has no such position: if interior code writes `UserId(...)`, it is
vouching for something it cannot know. That's not a style violation; it *is*
the bug from case (b), wearing a construction syntax.

**Three ways to make bad wraps much less likely** (Python can't make them
impossible):

1. **Remove the temptation rather than police it.** If interior functions
   *take* branded types as parameters, an interior developer is never
   holding a bare `int` that "needs" wrapping. If they somehow are, that's
   the design telling you to brand earlier — not an occasion to wrap at the
   error site.
2. **Concentrate the minting** in a few boundary modules — ideally one small
   factory module — so there is a single searchable target: "no `UserId(`
   outside boundary modules." That turns *silent wrap anywhere* into
   *auditable wrap at known sites*, the same greppability idea as
   `SafeHtml.assume_safe` in Point 2, and a reviewer can actually judge each
   one: case (a) or case (b)?
3. **Accept that this is convention, and notice why convention is enough
   here:** with no invariant, the worst a stray wrap can do is mislabel — a
   logic bug — not corrupt data or open an injection. Lower stakes, lighter
   tooling. (When there *is* an invariant, the stakes are corruption, and
   you spend the smart constructor.)

**The tell that you've outgrown the brand:** the moment you catch yourself
wanting to *validate at the mint* — to reject bad values while wrapping —
you have admitted the type has an invariant. "A brand that rejects bad
values" is a contradiction in terms; resolving it means promoting to a value
object with a smart constructor. That promotion is Point 2.

### 1d. Brands across languages: same idea, different guarantees

The concept is universal: give existing bits a new name, and have the
checker treat the new name as a different type so the two can't be confused.
Every statically typed language can express it. What differs — and what this
section compares — is the answer to four questions:

1. Is the new name **actually distinct** to the checker, or just an alias?
2. Does the distinction **survive at runtime**, or is it erased?
3. What does it **cost**?
4. Can you **control who mints** — or can anyone wrap any value, anywhere?

(All snippets below compiled and, where marked, run — rustc 1.90, go 1.26,
TypeScript 6.0 `--strict`, GHC 9.12. The "compile error" comments are actual
verified compiler rejections, quoted from output.)

**Rust** — a one-field "tuple struct" (the *newtype pattern*; see [Rust By
Example: New Type
Idiom](https://doc.rust-lang.org/rust-by-example/generics/new_types.html)):

```rust
struct UserId(u64);
struct ProductId(u64);

charge(u, p);                 // ok
charge(p, u);                 // compile error E0308: mismatched types
let raw: u64 = u.0;           // unwrapping is explicit: .0
let n: u64 = u;               // compile error E0308: no implicit conversion
```

A full nominal type: distinct in both directions, zero cost, and — unlike
any brand so far — the field can be made private, which restricts minting to
the defining module (more under Point 2).

**TypeScript** — structural typing means a plain alias
(`type UserId = number`) is *transparent*: it adds nothing (verified — both
directions assignable). To get distinctness you attach a phantom "brand"
property that exists only in the type:

```ts
type UserId    = number & { readonly __brand: "UserId" };
type ProductId = number & { readonly __brand: "ProductId" };

const u = 5 as UserId;        // the mint IS a cast -- there is no other way in
charge(u, p);                 // ok
charge(p, u);                 // compile error: brands are distinct
const n: number = u;          // brand -> base: implicit, fine
const v: UserId = 7;          // compile error: base -> brand needs the cast
```

Works, zero cost, fully erased at runtime — but note what minting looks
like: every mint is an `as` cast, and casting can never be restricted. The
convention burden of 1c applies double.

**Go** — a *defined type* (see the [Go spec: type
definitions](https://go.dev/ref/spec#Type_definitions)):

```go
type UserID int64
type ProductID int64

charge(u, p)                  // ok
charge(p, u)                  // compile error: cannot use p (ProductID) as UserID
var n int64 = u               // compile error: conversion must be explicit
_ = int64(u)                  // ...explicit in BOTH directions
_ = UserID(raw)               // ...but always available to anyone, anywhere
```

Distinct, zero cost, explicit both ways — and uniquely among these
languages, **the brand survives at runtime**: store a `UserID` in an `any`
and the interface remembers the defined type, so `i.(int64)` fails while
`i.(UserID)` succeeds (verified). A Go brand can actually be *checked* at a
runtime boundary, which no other language here can say.

**Haskell** — the original (`newtype`), with a capability the others lack:

```haskell
newtype UserId    = UserId Int      -- wrapper guaranteed erased at runtime
newtype ProductId = ProductId Int

charge u p     -- ok
charge p u     -- compile error: Couldn't match expected type 'UserId'
               --                with actual type 'ProductId'
```

Distinct, guaranteed zero cost — and because a module's export list controls
whether the `UserId` constructor is visible at all, **minting itself can be
restricted by the compiler**. Point 1c's whole "concentrate the mints,
enforce by grep" discipline becomes a module export in Haskell.

**The comparison:**

| Capability | Python `NewType` | Rust tuple struct | TS phantom brand | Go defined type | Haskell `newtype` |
| --- | --- | --- | --- | --- | --- |
| Distinct to the checker | yes | yes | yes (trick required; plain alias is transparent) | yes | yes |
| Survives at runtime | no — erased (`isinstance` raises) | no runtime need — compile-total | no — erased | **yes** — interfaces remember it | no — erased; compile-total |
| Runtime cost | zero | zero | zero | zero | zero (guaranteed) |
| Who can mint | anyone | anyone — or module-only via private field | anyone (`as` is always available) | anyone (conversion is always legal) | **export list decides** |
| Brand → base | implicit | explicit `.0` | implicit | explicit conversion | explicit unwrap |

Two cameos worth knowing about. **Scala 3** has this as a language feature:
[`opaque type UserId = Int`](https://docs.scala-lang.org/scala3/book/types-opaque-types.html)
is transparent inside its defining scope and distinct outside it, zero cost,
with minting controlled by what the companion object exposes — effectively
Haskell's capability with nicer ergonomics. And **F#** has [units of
measure](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/units-of-measure):
brands for numbers (`float<meters>` vs `float<feet>`) that the compiler
checks *through arithmetic* — dividing meters by seconds yields
`float<meters/seconds>` — then fully erases. That's the Mars Climate Orbiter
class of bug (Point 7) made unrepresentable.

The pattern in the table is the guide's Point 5 in miniature: everyone can
*name* the role; the languages differ in whether the name survives runtime
and whether the *mint* can be walled off — and those two capabilities are
exactly what Points 2 and 4 keep paying for in Python.

---

## Point 2 — Scenario: only some values are valid, and validity comes from work done while building

The situation. Some types have a real notion of a *bad value*, and the
difference between good and bad is some work that must be performed during
construction: HTML must be escaped, pointer tokens must be encoded, input
must be range-checked. Point 1 established that a brand can't help here. The
reflex is "make it a class" — so let's start with why that, by itself,
achieves nothing:

```python
class SafeHtml:
    def __init__(self, s: str) -> None:
        self._html = s          # accepts ANY string -- "safe" in name only
SafeHtml("<script>")            # constructs fine; no safer than a bare str
```

This has exactly the leak the brand had: the front door takes the *finished
form* — a string claimed to already be safe — and believes it. The class
added ceremony, not safety. What matters is never "is it a class"; it is
**"can the raw form come in the front door?"**

How to close the front door depends on one question about your domain:

> Do you ever legitimately need to accept an already-valid value that you
> *cannot rebuild from ingredients*?

"Ingredients" means the pre-work inputs — the token strings before joining,
the raw text before escaping. The two possible answers produce two designs,
called **Level A** and **Level B** throughout this guide.

### Level A — you can always rebuild from ingredients: remove the raw door entirely

JSON Pointer is the clean case. A pointer is entirely determined by its
tokens, and you always *have* the tokens (they're what you're trying to
point at). So there is never a reason for the public API to accept a
pre-formed pointer string. Let the constructor take only ingredients and do
the escaping itself:

```python
def pointer(*tokens: str) -> str:            # only ingredients; escaping internal
    ...
# pointer("axes", "x/y") -> "/axes/x~1y"     escaping applied for you
# pointer("a/b")         -> "/a~1b"          even MISUSE yields a VALID pointer
```

Pause on what "even misuse is safe" means. Suppose a caller wrongly passes
`"a/b"` thinking it's a path. They get `/a~1b` — a well-formed pointer to a
key literally named `a/b`. Wrong place, perhaps, but never a *malformed*
pointer, because "here is a finished string, trust it" simply is not part of
the public API. A caller can hand over wrong ingredients; they cannot hand
over a wrong *result*. There is no public way to express a bad value —
that's Level A.

Here is the same idea as a class. (Why bother with a class when the function
works? Because the function's output is a bare `str` — the moment it's
returned, nothing distinguishes it from any other string, and the guarantee
is lost. A class keeps the proof attached to the value as it travels, and —
see "closure" below — lets operations produce more proven values.)

```python
def _escape(t: str) -> str:
    return t.replace("~", "~0").replace("/", "~1")   # ~ first, or ~1's ~ gets re-escaped

class Pointer:
    __slots__ = ("_value",)

    def __init__(self, *tokens: str) -> None:        # ONLY ingredients
        self._value = "".join("/" + _escape(t) for t in tokens)

    def __truediv__(self, token: str) -> "Pointer":  # the "/" operator:
        new = Pointer.__new__(Pointer)                #   base / "axes" / name
        new._value = f"{self._value}/{_escape(token)}"
        return new

    def __str__(self) -> str:
        return self._value
```

Python mechanics, for readers who haven't met them: `__slots__` declares the
only attributes instances can have (mostly a memory/typo guard — more on its
limits below); `__truediv__` is how a class overloads the `/` operator, so
`Pointer("axes") / "x"` reads like a path; and `Pointer.__new__(Pointer)`
allocates an instance *without* running `__init__` — used here so the method
can extend an existing (already-escaped) value rather than rebuild from
tokens. That internal `__new__` doesn't violate our own front-door rule: the
rule governs the *public* surface, and inside the class we are the authority
maintaining the invariant ourselves.

(Verified: `Pointer("axes", "x/y")` gives `/axes/x~1y`; misuse
`Pointer("a/b")` gives the valid `/a~1b`; chained descent
`Pointer("axes") / "x/y" / "n~m"` gives `/axes/x~1y/n~0m`.)

### Level B — sometimes you're handed a finished value you can't rebuild: add one loud "trust me" door

`SafeHtml` cannot reach Level A, for a reason worth understanding rather
than memorizing. You will genuinely hold already-safe HTML with no
ingredients to rebuild from: the output of a sub-template, a constant
written by hand, two `SafeHtml` values concatenated. "Just re-escape it to
be sure" doesn't work, because escaping is not idempotent — escaping
already-escaped HTML corrupts it. Verified chain: `&` escapes to `&amp;`;
escape *that* and you get `&amp;amp;`, which a browser renders as the
literal text "&amp;". So a "this is already safe, take my word for it" path
is unavoidable. The design goal shifts: not to *eliminate* the unsafe path,
but to make it **loud, named, and impossible to take by accident**:

```python
class SafeHtml:
    __slots__ = ("_html",)

    def __init__(self, *_a: object, **_k: object) -> None:            # naive path blocked
        raise TypeError("use SafeHtml.escape(...) or SafeHtml.assume_safe(...)")

    @classmethod
    def escape(cls, raw: str) -> "SafeHtml":                          # DEFAULT, safe
        obj = object.__new__(cls)
        obj._html = raw.replace("&", "&amp;").replace("<", "&lt;")
        return obj

    @classmethod
    def assume_safe(cls, already_safe: str) -> "SafeHtml":           # explicit, greppable hatch
        obj = object.__new__(cls)
        obj._html = already_safe
        return obj

    def __str__(self) -> str:
        return self._html
```

(Same `object.__new__` mechanic as before: it creates the instance without
running `__init__`, which is how the two classmethods can build values while
the ordinary `SafeHtml(...)` call is booby-trapped to refuse.)

Why this is meaningfully safer, even though `assume_safe` will still accept
an unescaped string (all three behaviors verified):

- **The safe path is the default and the obvious one.** The method you reach
  for by habit — `escape` — is the correct one.
- **The naive path fails fast.** `SafeHtml("<script>")` raises immediately,
  so nobody gets an unsafe value by "fixing a type error the lazy way" — the
  exact move dissected in Point 1c.
- **The unsafe path is named as an assertion.** Nobody types `assume_safe`
  by accident — the name says what you're claiming — and `grep assume_safe`
  lists every trust decision in the codebase in one command. The danger is
  concentrated at a handful of reviewable sites instead of smeared invisibly
  across every construction.

**The reframe to take away: "airtight" is not a yes/no property.** It's a
spectrum of *how loud and auditable the unsafe path is*:

- **Level A** (Pointer): the unsafe path **does not exist** — everything is
  rebuilt from ingredients. Maximum airtight.
- **Level B** (SafeHtml): the unsafe path **must exist**, so it is named and
  greppable. Airtight against accidents; auditable for the deliberate cases.
- **Brand / broken-SafeHtml**: the unsafe path **is the default front
  door**, and using it looks like normal code. Silent, unauditable — the
  level to avoid.

Which level you can reach is dictated by the domain, not by taste: can you
always rebuild from ingredients (Level A), or must you sometimes accept a
pre-made value (Level B)?

### "Why can't we just check the finished value?" — the bug that leaves no trace

For some types you genuinely can check afterward — that's Point 3's
recoverable branch. This section shows why JSON Pointer is not one of them,
because the failure mode is instructive: **the bug produces a valid value.**

The invariant is "every `/` or `~` inside a token's content was escaped."
Watch it being violated:

- Correct: intended tokens `["axes", "x/y"]`, escaping applied, result
  `/axes/x~1y`.
- Buggy: same intended key `"x/y"`, escaping forgotten, result `/axes/x/y`.

Now put yourself in a validator's shoes, looking at `/axes/x/y` with no
other context. It's a perfectly well-formed pointer — it means three tokens,
`["axes", "x", "y"]`. Was that what the producer meant? Or did they mean two
tokens, one containing a slash, and forget to escape? **Both stories end in
valid strings, and the string does not record which story happened.** The
forgotten escape didn't corrupt the pointer's format; it silently changed
the pointer's *meaning* — from "the key `x/y` under `axes`" to "the key `y`
under `x` under `axes`".

That's what this guide means by **unrecoverable**: no check on the finished
value can catch the mistake, because nothing about the finished value is
wrong — the only thing wrong is a mismatch with what the producer intended,
and intent is exactly the information the finished value doesn't carry. The
only defense is to make the mistake impossible to *commit*: build from
tokens (Level A), where escaping is applied by the constructor and the
meaning-corrupting `/axes/x/y` simply cannot be produced through the API.

### Built right, then changed: construction control needs immutability

Controlling construction is necessary but not sufficient. A value built
correctly can still be *edited* into invalidity afterward — and Python's
`__slots__`, which the classes above use, does **not** prevent that.
`__slots__` limits *which* attributes exist; it does not make them read-only
(verified — this runs without complaint):

```python
s = SafeHtml.escape("safe")
s._html = "<script>"          # no error -- invariant broken AFTER construction
```

Three things close the gap:

1. **Store immutable underlying data** (`str`, `tuple` — things Python
   won't let anyone modify in place), rather than mutable ones (`list`,
   `dict`).
2. **Expose no mutators, and never leak an internal mutable reference.** If
   you stored a `list` and a method returns it directly, callers now hold a
   handle to your insides and can edit them. Return a copy, or store a
   `tuple` in the first place.
3. **Make the wrapper's own fields read-only** with frozen semantics:
   `@dataclass(frozen=True)` (verified: reassigning a field raises
   `FrozenInstanceError`), a frozen msgspec Struct, or a custom
   `__setattr__` that raises.

The principle: **airtight = controlled construction *and* immutability.**
Skip either and the invariant leaks — through the front door or through the
window.

### The constructor is safe, but a method leaks: operations must stay in the type

In practice the most common leak isn't the constructor at all. It's a
convenience method, added months later, that hands back the raw form:

```python
class Pointer:
    def with_suffix(self, s: str) -> str:      # LEAK: hands back a raw str;
        return str(self) + s                   #   the guarantee ends here
    def __truediv__(self, token: str) -> "Pointer":  # CLOSED: returns Pointer,
        ...                                          #   escapes the token itself
```

The principle is called **closure** (in the algebra sense — operations on
the type produce values still in the type): every operation should take the
refined type and return the refined type, re-establishing the invariant
internally. `SafeHtml + SafeHtml` should give `SafeHtml`; `Pointer / token`
should give `Pointer`. A method that returns the raw base type — or accepts
one and trusts it — undoes the constructor's work one call at a time. This
is where "safe constructor, leaky method" bugs live.

### Three more things worth knowing

- **The payoff — why go to all this trouble:** once the type is airtight,
  downstream code never re-checks. Holding a `Pointer` *is* the proof; no
  function that receives one needs a defensive re-validation. You pay for
  correctness once, at construction; every consumer gets it free forever.
  This is the productivity argument for the whole approach, and it is
  exactly what "parse, don't validate" means.
- **Builders that can't fail vs. builders that can say no:**
  `pointer(*tokens)` is *total* — every input produces some valid pointer
  (even misuse produces valid-but-wrong), so callers have no error case to
  handle. A validating `Percentage(v)` is *partial* — it can raise. Prefer
  total when the domain allows it (no error handling to thread through every
  caller); use partial when bad input genuinely must be rejected. A "parser"
  is precisely a partial constructor: produce the refined type or fail.
- **The hatch-naming convention is established, not invented here:**
  `assume_safe`, `_unchecked`, `unsafe_` follow the pattern of Rust's
  `str::from_utf8_unchecked`, `get_unchecked`, and the `unsafe` keyword
  itself: safe API as the default, bypass loudly labeled. Level B is a
  recognized industry pattern.

### The same fortress in five languages

The design is identical everywhere — hide the raw way in, export builders
that do the work — and it originates in Haskell, where it's simply called
the [smart constructor idiom](https://wiki.haskell.org/Smart_constructors).
What differs per language is **who guards the wall**: the compiler, the
runtime, or convention. (All snippets compiled here; each "error" comment is
a verified rejection.)

**Haskell** — the canonical form. A module exports the *type* but not its
*data constructor*, and the compiler does the rest:

```haskell
module Pointer (Pointer, pointer, render) where   -- P is NOT in the list

newtype Pointer = P String

pointer :: [String] -> Pointer        -- the only public builder; escapes internally
pointer = P . concatMap (('/' :) . escape)

-- outside this module:
--   P "raw"   -->  error: Data constructor not in scope: P
```

**Rust** — same idea, spelled with field privacy:

```rust
pub struct Pointer { value: String }        // field is private to the module
impl Pointer {
    pub fn new(tokens: &[&str]) -> Pointer { /* escapes, joins */ }
}
// outside the module:
//   Pointer { value: "raw".into() }
//   --> error[E0451]: field `value` of struct `Pointer` is private
```

**Go** — same idea, spelled with an unexported field; the wall stands at the
package boundary:

```go
type Pointer struct{ value string }         // lowercase = unexported

func New(tokens ...string) Pointer { /* escapes, joins */ }
// outside the package:
//   pointer.Pointer{value: "raw"}
//   --> cannot refer to unexported field value in struct literal
// BUT this compiles anywhere:
//   pointer.Pointer{}                      // the zero-value loophole
```

Go's caveat is that loophole: every Go type has one constructor you cannot
remove — the zero literal. `pointer.Pointer{}` builds a `Pointer` whose
value is `""` outside any constructor (verified). Hence the Go proverb "make
the zero value useful": design the type so the zero value is *valid* (for
JSON Pointer, `""` happens to mean "whole document" — legitimately valid;
a `Percentage` with a minimum of 1 would not be so lucky).

**TypeScript** — a class with a `private constructor` (compile-time) and an
ES `#private` field (real runtime privacy — unlike TS's `private` keyword or
Python's underscore convention, `#value` is enforced by the JavaScript
engine itself):

```ts
class Pointer {
  readonly #value: string;
  private constructor(value: string) { this.#value = value; }
  static fromTokens(...tokens: string[]): Pointer { /* escape, join */ }
}
// new Pointer("raw")            --> compile error: constructor is private
// { toString: () => "raw" }     --> compile error: missing #value
```

That second rejection is worth a pause: normally TypeScript's structural
typing lets any object with the right shape impersonate a class. A
`#private` field cannot be declared outside its class, so no impersonation
can ever have one — **a `#private` field quietly makes the class nominal**
(verified), closing the forgery hole that plain branded types leave open.

**The comparison:**

| Language | Who blocks the raw path | Wall's scope | Immutability story |
| --- | --- | --- | --- |
| Haskell | compiler (module export list) | module | everything is immutable, always |
| Rust | compiler (private field) | module | immutable by default; `mut` is opt-in and visible |
| Go | compiler (unexported field) | package — minus the zero-value loophole | no read-only fields; rely on unexported + no setters + value-copy semantics |
| TypeScript | compiler (`private constructor`) **and** runtime (`#private`) | class | `readonly` is compile-only; `#private` + no setters is the runtime floor |
| Python | convention + runtime tricks (this whole Point) | none — discipline, plus greps | `frozen=True` at runtime; bypassable via `object.__setattr__` |

Reading the table top to bottom is reading the guide's difficulty curve in
reverse: in Haskell and Rust, Level A is a handful of lines and the compiler
guarantees it; in Go it's real but has a known loophole; in TypeScript it's
surprisingly strong at runtime thanks to `#private`; in Python every
guarantee in this Point had to be assembled by hand and still tops out at
"airtight against accident."

---

## Point 3 — Scenario: a new type is in front of you — which treatment does it need?

Points 1 and 2 each handled one situation. This point is the map: given any
candidate type, two **questions** tell you which treatment it needs, and two
**overlays** adjust the construction. ("Overlay" because they are not steps
3 and 4 of a sequence — each applies independently, on top of whichever
branch the questions chose.)

```
Q1. Are only SOME base values valid?  (Is there an invariant?)
    NO  -> NewType / brand (Point 1).  Done.
    YES -> Q2.

Q2. Is validity a decidable property of the FINISHED value alone?  (recoverable?)
    YES -> validating constructor: accept the finished form, check, raise.
           (partial; this IS a parser.       Percentage, DateRange)
    NO  -> validity is a fact about HOW the value was produced ->
           construct-only: accept INGREDIENTS, never the finished form.
           (pointer(*tokens), SafeHtml.escape)

Overlay 1 — boundary (any branch): can values arrive from untyped / JSON /
    public callers?  YES -> a runtime guard as well; static types don't fire
    there (Point 4).  (On the Q2-YES branch the validating constructor
    already IS the guard — just route boundary input through it.)

Overlay 2 — hatch (any branch): must you accept an already-valid value you
    can neither rebuild from ingredients nor (re)check?
    NO  -> no raw path at all.                       (Level A — Pointer)
    YES -> named unsafe hatch, safe default.         (Level B — assume_safe)
```

The rest of this point walks the tricky parts: a trap hiding inside Q1, the
precise meaning of Q2, a litmus test for hard Q2 calls, a subtlety about
direction, and why the "trust me" door shows up on both branches.

### "But a UserId must be a *real* user" — rules about the world are not rules about the value

A trap waiting at Q1: it's easy to talk yourself into an invariant that
isn't one. "Surely `UserId` has a validity rule — it has to identify a user
that actually exists!"

Look closely at what that rule refers to. "Row 5 exists in the users table"
is a fact about *the database, right now*. The user can be deleted while
your `UserId` sits in a queue — and when that happens, the int in your hand
doesn't change. Nothing about the *value* went bad; the *world* moved. A
type can only carry **timeless** facts: properties of the value itself that
hold forever, wherever it travels. "0 ≤ v ≤ 100" is timeless. "Exists in
the DB" is not.

If you promote `UserId` to a validating class because of a world-fact, you
buy a check that is stale the instant it finishes — and worse, a type whose
name now *claims* a guarantee it cannot keep, which is the false-confidence
trap of Point 6. Existence is a *query*: ask it at the moment you need it,
at the place you need it. It is not an invariant, and no constructor can
make it one.

### The fork that decides everything: can validity be checked later, or is it lost at build time?

Q2 is the load-bearing question of the whole guide, so here are the two
answers as precise definitions:

> **Recoverable:** validity is a yes/no question you can answer by examining
> the finished value alone — anyone can re-check it, at any time. For these
> types, *checking* a value is exactly as good as *controlling* how it was
> built.
>
> **Unrecoverable:** validity depends on the value's **history** — what the
> producer meant, where the value came from — and the finished value does
> not record its history. No examination of the value can settle it.
> Construction is the *only* moment the guarantee can be made.

Run the examples through the definitions:

- `Percentage`: "0 ≤ v ≤ 100" mentions only the value. Anyone can re-check
  at any time → recoverable → a validating constructor is enough.
- `Pointer` (being built): "were these three tokens, or two with a forgotten
  escape?" mentions *what the producer meant* → unrecoverable.
- `SafeHtml`: "did this markup come from our template or from a user?"
  mentions *where the value came from* → unrecoverable.

Notice the pattern: both unrecoverable examples are questions about the
producer, not about the bytes. That is the general shape, and it's why
unrecoverable invariants and the provenance family (Point 7) are the same
thing: **an unrecoverable invariant *is* a fact about origin.**

One clarification that trips people up: a rule involving *several* fields is
still recoverable — "start before end" mentions two fields, but both are
right there in the value, so anyone can still check it whenever
(verified: raises on bad input; frozen against later edits):

```python
@dataclass(frozen=True)
class DateRange:
    start: date
    end: date
    def __post_init__(self) -> None:          # runs right after the generated
        if self.start > self.end:             #   __init__; cross-field, but still
            raise ValueError(f"start {self.start} after end {self.end}")  # checkable
```

(`__post_init__` is the dataclass hook that runs immediately after the
auto-generated constructor assigns the fields — the natural place for a
validating check.)

### Two identical strings, one valid and one not: the litmus test for hard cases

Q2 can be genuinely hard to answer, and `SafeHtml` shows why — its invariant
*looks* checkable. "Reject any string containing a raw `<`" is a perfectly
implementable check. But it's the wrong check: your own template's output
legitimately contains raw `<b>` tags. So consider two values:

- `<b>hi</b>`, produced by your template: **valid** — it's your markup.
- `<b>hi</b>`, typed by a user into a comment box: **an injection** — user
  text was supposed to arrive escaped.

The two strings are byte-for-byte identical. No check that looks only at the
string can tell them apart — there is nothing *in* the string to see. What
differs is where each came from. Hence the litmus test for hard Q2 calls:

> **If two identical values can differ in validity, no checker can exist,
> and the invariant is unrecoverable** — the type has to carry a fact that
> the value itself doesn't.

(This is the same shape as `/axes/x/y` in Point 2. There the invisible
difference was *intent*; here it's *trust*. Both are history.)

### Building a pointer vs. receiving one: the same type can sit on both sides

A subtlety the flat tree hides: for one type, *producing* values and
*consuming* them can land on different Q2 branches.

- **Building** a pointer out of key strings is the unrecoverable case we
  know: once the tokens are joined, "two tokens, escaped" and "three tokens"
  look identical. So building goes through ingredients — `pointer(*tokens)`,
  Level A.
- **Receiving** a pointer is a different situation. When a config file or a
  JSON document hands you `"$ref": "#/definitions/x"`, there is no
  "producer's intent" left to worry about — *the string itself is the
  authority now*. A received `/axes/x/y` means three tokens, by definition;
  if the document's author meant something else, that was their bug, on
  their side of the wire, and no amount of care on our side can see it. All
  that's left for us to check is *grammar* — does it start with `/`? is
  every `~` a proper `~0` or `~1`? — and grammar questions are answerable by
  looking at the string. Recoverable! So a checking, rejecting `parse` is
  legitimate here — and necessary, because wire pointers really do arrive
  (verified; this is a method of the `Pointer` class above):

```python
_TOKEN = re.compile(r"^(?:[^~/]|~[01])*$")     # only ~0 / ~1 escapes allowed

@classmethod
def parse(cls, s: str) -> "Pointer":           # for RECEIVED pointers (partial)
    if s == "":
        return cls()                           # whole document
    if not s.startswith("/"):
        raise ValueError(f"pointer must start with '/': {s!r}")
    tokens = s.split("/")[1:]
    if any(not _TOKEN.match(t) for t in tokens):
        raise ValueError(f"bad escape in {s!r}")   # bare ~, ~2..~9
    # RFC 6901: unescape ~1 -> / FIRST, then ~0 -> ~
    # (verified: reverse order corrupts token "~1", escaped "~01", into "/")
    return cls(*(t.replace("~1", "/").replace("~0", "~") for t in tokens))
```

So `Pointer` legitimately has **two doors**: `parse(str)` — the checking
door, for the boundary, for wire strings whose meaning is already fixed —
and `pointer(*tokens)` — the building door, for the interior, for meanings
you are fixing right now. They agree with each other by the round-trip law
`parse(pointer(*t)).tokens == t` (verified).

What must **not** exist is the hybrid: building a pointer by gluing raw keys
together (`"/axes/" + key`) and treating the result as if it had been
received. That is *building* through the *receiving* door — with
`key = "x/y"` it manufactures exactly the meaning-corrupting `/axes/x/y` —
the unrecoverable bug in one line of string concatenation. Rule of thumb:
`parse` is for strings whose meaning is already fixed; ingredients are for
meanings you're fixing now.

### The "trust me" door exists for two different reasons

| Branch | Why a hatch | Example | Cost of misuse |
| --- | --- | --- | --- |
| Unrecoverable | **necessity** — no check is possible, so trusted pre-made values can only be *asserted* | `SafeHtml.assume_safe` | invariant silently broken |
| Recoverable | **performance** — the check exists but is O(n)/hot-path; skip it for a trusted producer | Rust `str::from_utf8_unchecked` | same, plus it was avoidable |

Rust's standard library shows the recoverable branch grows hatches too:
UTF-8 validity *can* be checked (that's what `from_utf8` does), and
`from_utf8_unchecked` exists purely to skip the cost of re-checking when the
caller already knows. Same mechanism as `assume_safe` — a named, greppable
bypass — but a different reason for existing, and the difference matters. On
the recoverable branch the hatch is always *optional*: when in doubt, just
take the checking door. On the unrecoverable branch the hatch is
*load-bearing*: remove `assume_safe` and legitimately pre-made values — the
template's output — have no way into the type at all.

### The running examples, walked through the tree

| Type | Q1 invariant? | Q2 recoverable? | Overlay 1 (boundary) | Overlay 2 (hatch) | Result |
| --- | --- | --- | --- | --- | --- |
| `UserId` | no | — | convention: mint at boundary (Pt 1c) | — | `NewType` |
| `Percentage` | yes | yes — predicate of value | constructor doubles as guard | none needed | validating ctor |
| `DateRange` | yes (cross-field) | yes | same | none needed | frozen dataclass + `__post_init__` |
| `Pointer` (producing) | yes | **no** — intent | — | no: always rebuildable | Level A, from tokens |
| `Pointer` (receiving) | yes | grammar: yes | `parse` at boundary | — | partial `parse` |
| `SafeHtml` | yes | **no** — trust | — | yes: can't rebuild (double-escape) | Level B + `assume_safe` |

---

## Point 4 — Scenario: the edges of your program, where the type checker can't see

Everything so far leaned on the type checker — and a type annotation is
**not** a runtime check. It's a note to the checker, and by itself it does
nothing while the program runs. (That is precisely why the
runtime-validation ecosystem exists: msgspec decoding, pydantic, and
beartype all *read* annotations and enforce them — but each only at the
specific place where it's installed.) So a static guarantee holds exactly as
far as the checker can see:

```python
@dataclass
class Issue:
    at: Pointer

Issue(at="literally anything")    # runs fine -- annotations do nothing at runtime
```

One piece of vocabulary for this point: a **seam** is any line where code
with one set of guarantees meets code with another — type-checked interior
against untyped caller, decoded objects against raw wire bytes, a specific
type against an `Any`. The checker's sight ends at seams. It has **four
distinct kinds** of them — four holes, all verified — and the punchline of
the whole point is that runtime checks belong exactly at the seams and
nowhere else:

**Hole 1 — callers who don't run the checker.** Public API consumers,
scripts, notebooks, plugins, teammates who haven't set up mypy. If the
caller's code was never type-checked, your annotations were never consulted.
Organizational, obvious — and still the most common hole in practice.

**Hole 2 — deserialization and DTO construction.** Whether constructing a
data object is checked at runtime is a *per-library design choice*, and the
ecosystem genuinely splits — know which side your library is on (all
verified):

- **msgspec** validates the wire shape *on decode* — `{"at": 5}` is rejected
  for a `str` field — but `Issue(at=5)` constructs silently: the decode call
  is the only guarded seam.
- **dataclasses, attrs (by default), TypedDict** (a plain dict at runtime)
  check nothing anywhere.
- **pydantic** validates on *direct construction too*: `Issue(at=5)` raises
  `ValidationError`. A `BaseModel` is a validating constructor — Point 3's
  Q2-YES branch packaged as a library, and much of why pydantic is popular.
  And true to Point 2, its bypass is a *named, greppable* Level B hatch:
  `Issue.model_construct(at=5)` skips validation silently, by design.

The trap is assuming your DTO layer sits on pydantic's side when it sits on
msgspec's: then nothing stops interior code from constructing `Issue(at=5)`
directly, and the annotation's promise is empty everywhere except the decode
call.

**Hole 3 — `Any` poisoning *inside* fully-checked code.** The first two
holes are about *who* runs the checker; this one is about the checked code
itself. `Any` is the type checker's "anything goes" type: an expression of
type `Any` is assignable to a variable of *any* type, no questions asked.
And `Any` appears in ordinary, innocent-looking code — most famously,
`json.loads` returns it:

```python
a: UserId = json.loads("5")        # json.loads returns Any  -- mypy: silent
b: UserId = from_api()             # any "-> Any" function   -- silent
c: UserId = cast(UserId, "junk")   # cast is an unchecked lie -- silent
```

Verified (mypy 2.1, defaults): all three pass. So the boundary is not just
organizational — it is **textual**: wherever `Any` appears, the checked
region has an interior hole.

Can strictness flags close it? Surprisingly, no — and the reason is design,
not accident. Even `--disallow-any-expr`, the strictest Any flag, stays
silent on all three lines above, because per mypy's own docs an `Any`
expression errors "*unless the expression is immediately used as an argument
to `cast()` or assigned to a variable with an explicit type annotation.*"
Verified across mypy 1.15 and 2.1: `Any` used as an argument, as an operand,
or in an *unannotated* assignment is flagged, while the `cast(...)` and
annotated-assignment forms are exempt. In other words, the flag polices the
accidental *spread* of `Any` — but the deliberate *claim* is sanctioned.

Now read those two exemptions through this guide's own taxonomy: they are
the static system's escape hatches, and they land on opposite levels.
`cast(UserId, x)` is a **named, greppable hatch** — Level B, auditable with
`grep cast(`, exactly like `assume_safe`. But `a: UserId = json.loads(...)`
is a **silent hatch**: it launders `Any` into any refined type while looking
exactly like ordinary well-typed code — the broken-SafeHtml shape, a front
door dressed as a wall. The conclusion: even maximal static strictness
terminates in honor-system assertions at every `Any` seam, which is why the
only *checking* device at such a seam is a runtime one (parse or guard,
below).

**Hole 4 — brands have no runtime identity at all.** `UserId(5)` *is* `5`.
There is no runtime artifact to test for: `isinstance(x, UserId)` doesn't
return `False` — it **raises `TypeError`** (verified), because `UserId`
isn't a class. The only thing a runtime check *can* test is the base type
(`isinstance(x, int)`), and that proves the value is an int — its
*representation* — never that it plays the user-id *role*. Which is Point 3
resurfacing at runtime: a brand's fact — "this int came from the user table"
— is a fact about origin, and origin can't be checked into existence after
the fact. At a boundary, a brand can only be **re-minted by an authority**
(Point 1c): re-parsed, re-looked-up — never `isinstance`d.

### At the seam, two different jobs: parse the raw, guard the "finished"

What a seam should do depends on what the arriving value claims to be:

- **It arrives raw** — say, a `str` that denotes a pointer. The job is to
  **parse**: route it through the checking door (`Pointer.parse`, Point 3),
  whose whole contract is to transform raw input into the refined type or
  reject it. (For the refined type itself no extra machinery is needed — the
  smart constructor already *is* the runtime check.)
- **It claims to be refined already** — the field says `Pointer`, but the
  value crossed one of the four holes to get here, so the claim is
  unverified. The job is to **guard**: verify the claim, and reject if it's
  false:

```python
@dataclass(frozen=True)
class Issue:
    at: Pointer
    def __post_init__(self) -> None:
        if not isinstance(self.at, Pointer):    # catches a raw str at runtime
            raise TypeError("at must be a Pointer built from tokens")
```

A guard **rejects**; it never "fixes." Worth knowing: the frozen dataclass
does not actually enforce that discipline — `object.__setattr__` inside
`__post_init__` can still overwrite fields (verified), so "reject, don't
repair" is a convention the frozen idiom *encourages*, not a wall it builds.
The convention matters because the two jobs have different contracts.
A parser's contract is "input is raw; transforming it is my job." A guard's
contract is "this value should already be right; I only verify." A guard
that quietly repairs is absorbing exactly the bugs it was posted to catch —
you wanted to hear about those.

### What does `isinstance` really prove? The subclass loophole

`isinstance(x, Pointer)` implies "this went through the smart constructor"
only while two things hold: the class is airtight (Point 2), **and no
subclass lies**. The second condition is real — a subclass can inherit the
type's identity while skipping its construction discipline (verified):

```python
class Evil(Pointer):
    __slots__ = ()
    def __init__(self) -> None:        # never calls super()
        self._value = "raw/unescaped"

isinstance(Evil(), Pointer)            # True -- invariant broken
```

The instinctive fix is `@typing.final` (a marker meaning "no subclassing
allowed") — but finality is *itself* a static-only guarantee that evaporates
at runtime, which is this very section's principle applied to its own
machinery. The runtime backstop is `__init_subclass__` raising — a hook
Python calls whenever a subclass is being defined, so the forgery fails at
class-definition time (verified). Proportionality (Point 6) governs the
choice: seal the class when the guard is load-bearing for security;
otherwise "don't subclass" in the docstring is the right spend.

Same theme, fine print: anything that rebuilds objects via `__new__` plus
state restore — `copy`, `pickle`, some ORMs and decoders — skips `__init__`
and `__post_init__` entirely (verified), so no constructor guard runs on
those paths. As with Python privacy (Point 5), the honest framing of the
threat model: these guards defend against *accident, not malice*.

### Where the checks go: one guard per seam, not one per function

If interior code re-checks refined values everywhere "just in case," you
have quietly re-invented correct-by-validation — scattered checks, paid on
every call — and forfeited Point 2's payoff, which was that holding the type
*is* the proof. The guard belongs exactly where a value crosses from the
unchecked world into the checked one. That is the same seam where brands are
minted (Point 1c): the decode call, the request parser, the public entry
point. Cross once, check once; inward of the seam, trust the type.

For a public library API too wide to hand-guard, decorator-based enforcers
(beartype, typeguard) install the same isinstance discipline systematically
at the entry points — still one check per seam, just automated.

The division of labor, in one line: **static types guard the interior;
runtime guards stand at the seams — wire data, unchecked callers, and every
`Any`; brands can't be guarded at all, only re-minted.**

### The four holes, across languages

The hole *pattern* is universal; which holes actually exist — and how wide —
is one of the sharpest differences between languages.

**Hole 1 — unchecked callers.** A Python or TypeScript library faces callers
who skipped the checker (mypy is optional; plain JavaScript happily calls
TS-compiled code). In Rust, Go, and Haskell this hole does not exist inside
a program: compilation *is* the type check, and nobody links code that
failed it. The hole reappears only at the language border — C callers over
FFI, cgo — which is why those ecosystems talk about `unsafe`/FFI hygiene
instead of caller discipline.

**Hole 2 — decode.** Exists everywhere, with different escape routes:

- **Go:** `json.Unmarshal` writes exported struct fields directly — no
  constructor runs, and missing fields silently become zero values
  (verified). Decoding is a second zero-value loophole.
- **Rust:** serde's derived `Deserialize` also constructs directly — but
  `#[serde(try_from = "RawType")]` routes decoding through your validating
  constructor, making parse-at-the-seam a one-line attribute (documented in
  serde's container attributes).
- **TypeScript:** `JSON.parse` returns `any` (verified below) — the decode
  hole and the `Any` hole are the *same hole*, which is why the ecosystem
  grew parse-at-the-seam libraries (zod, io-ts) whose entire job is Point
  3's "validating constructor" for wire data.
- **Haskell:** aeson's `FromJSON` is an instance *you write*, so routing it
  through the smart constructor is the natural spelling, not an add-on.
- **Python:** the per-library split documented above (msgspec vs pydantic).

**Hole 3 — the `Any`-shaped type.** Python's `Any` and TypeScript's `any`
are near-identical poisons (TS: `const u: UserId = JSON.parse("5")` compiles
clean under `--strict` — verified). TypeScript's partial antidote is
`unknown`, the "safe any": it accepts anything but must be narrowed before
use (`const x: UserId = someUnknown` is a verified compile error — Python's
closest analog is `object`). Go's `any` looks similar but is much narrower:
getting a value back out requires a type assertion `v.(T)`, which is
**checked at runtime** — no silent laundering. Rust and Haskell have no safe
equivalent at all; Rust's `transmute` exists but is `unsafe` — a named,
greppable, Level B hatch rather than a silent one.

**Hole 4 — brand erasure.** Python: erased (`isinstance` raises — verified).
TypeScript: erased, and `as` can mint anywhere. **Go: not erased** — an
interface remembers the defined type (`i.(int64)` fails, `i.(UserID)`
succeeds — verified), so a Go brand can genuinely be guarded at a runtime
seam. Rust and Haskell erase brands at runtime too, but their compile
coverage is total, so no *interior* seam ever needs the check — the only
seams left are decode and FFI, already covered.

The roll-up: **the weaker a language's static reach, the more seams need
runtime guards.** Rust and Haskell funnel every seam into decode + FFI;
Go adds the zero value and a narrow, checked `any`; TypeScript is erased at
runtime everywhere except `#private` — hence its parse-library culture;
Python has the most seams of all — hence this Point.

---

## Point 5 — The capability matrix: what each language actually gives you

Points 1d, 2, and 4 each closed with a cross-language comparison of one
concept. This point is the roll-up — every row below is verified in the
section it references (OCaml sits with Haskell throughout: its `.mli`
interface files hide constructors the same way Haskell's export lists do):

| Capability | Python | TypeScript | Go | Rust | Haskell |
| --- | --- | --- | --- | --- | --- |
| Brand distinct to the checker (1d) | ✔ `NewType` | ✔ via phantom brand | ✔ defined type | ✔ tuple struct | ✔ `newtype` |
| Brand survives at runtime (1d, Hole 4) | ✘ erased | ✘ erased | **✔ interfaces remember it** | compile-total | ✘ erased; compile-total |
| Restrict who mints a brand (1d) | ✘ convention | ✘ convention (`as` always available) | via one-field struct | ✔ private field | ✔ export list |
| Block raw construction (Pt 2) | ✘ convention + runtime tricks | ✔ compile + **runtime `#private`** (also defeats structural fakes) | ✔ package boundary — minus the zero-value loophole | ✔ module boundary | ✔ module boundary |
| Immutability (Pt 2) | runtime `frozen` (bypassable) | `readonly` compile-only; `#private` floor | ✘ no read-only fields; copies + discipline | ✔ default | ✔ always |
| Decode routes through your constructor (Pt 4, Hole 2) | per-library (pydantic yes, msgspec decode-only) | ✘ `JSON.parse` is `any`; zod at the seam | ✘ `Unmarshal` bypasses; zero values for gaps | opt-in `#[serde(try_from)]` | ✔ you write `FromJSON` |
| `Any`-poison exposure (Pt 4, Hole 3) | high (`Any`, with laundering exemptions) | high (`any`) but `unknown` mitigates | narrow — assertions checked at runtime | ~none safe (`transmute` is `unsafe`, greppable) | ~none |

The judgment this buys, language by language:

- **Rust / Haskell (and OCaml):** the pattern is **cheap and total** — a
  handful of lines, compiler-guaranteed, zero cost. Reach for it freely;
  there is little reason to settle for a bare brand when the fortress is
  this cheap. All runtime worry funnels into two seams: decode and FFI.
- **Go:** strong walls at package boundaries; budget your care for the two
  loopholes (zero values, `Unmarshal`) — and enjoy the one superpower nobody
  else has: brands that can be `isinstance`d, so a runtime guard can check
  *role*, not just representation.
- **TypeScript:** compile-time story is strong and the `#private` runtime
  story is underrated — but everything else erases, and decode is an `any`
  firehose. The parse-at-the-seam culture (zod) is not optional hygiene;
  it's the load-bearing guard.
- **Python:** every capability in the matrix is partial and costs a runtime
  check. That doesn't make the pattern worthless — it makes it something you
  **spend deliberately** where the stakes warrant (Point 6), with honest
  expectations: airtight against accident, auditable against intent.

---

## Point 6 — Proportionality: match the guard to the stakes

The trap on one side is the flimsy guard; on the other, the reflex "always
build the value object." Four factors set the right spend:

- **Blast radius** of one bad value: silent corruption / injection -> heavy;
  a wrong diagnostic string -> light.
- **Producer breadth:** one reviewed module -> light; public or third-party
  producers -> heavy.
- **Caller type-checking:** interior + mypy -> a static brand may carry the
  weight; untyped / serialized ingress -> needs the runtime guard (Point 4).
- **Recoverability** (Point 3 Q2): unrecoverable -> no choice; construction
  control is the only mechanism that can work.

| Situation | Enough |
| --- | --- |
| Internal, one module, type-checked callers, recoverable invariant | a brand — or even a naming convention |
| Unrecoverable invariant, OR external producers, OR an untyped/JSON boundary, OR a dangerous bad value | smart constructor + runtime guard |

**The anti-pattern to actively avoid: a guard that looks stronger than it
is.** A brand on a public surface gives *false confidence* — a reviewer sees
the refined type and stops scrutinizing, while the front door still accepts
garbage. False confidence is worse than no guard: it moves the mistake from
visible to invisible.

---

## Point 7 — The provenance family (and the one heuristic to carry)

Point 3 established that an unrecoverable invariant is a fact about a value's
*history*. Types that carry such a fact are everywhere, and they are all the
same shape — **the type encodes "this value was produced by process X," which
no post-hoc check can recover**:

- **Sanitization:** `SafeHtml` / `SqlFragment` vs raw `str` — literally why
  HTML-escaping libraries, SQL tagged-template APIs, and Python 3.14 t-strings
  (PEP 750) exist. The type *is* the provenance.
- **Taint:** `Untrusted[str]` vs `Trusted[str]` at trust boundaries.
- **Units:** `Meters` vs `Feet` — the Mars Climate Orbiter tax. The float
  `3.0` doesn't record its unit any more than a string records its escaping.
- **Normalization:** `LowercasedEmail`, `AbsolutePath`, `NfcString`.

In every case validity is established at construction and unreadable
afterward, so the architecture must be "control the constructor," and the
type records something the bytes alone can't say.

### The one heuristic to carry

> **First ask whether the invariant is checkable after the fact.** If yes,
> easy options exist (a validating constructor — or, with no invariant at all,
> a brand). If no, you **must** own construction — the public constructor
> takes ingredients, never the finished form — plus a runtime guard at any
> boundary the type checker doesn't cover. Everything else (brands, aliases,
> conventions) is for the easy, recoverable, internal cases.
