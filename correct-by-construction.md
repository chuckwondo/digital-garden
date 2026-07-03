# Correct by Construction

*Refined types, smart constructors, and where invariants come from.*

A general architecture guide (language-agnostic in spirit; examples in Python).
The thesis: when only *some* values of a type are valid, the guarantee that a
value is valid should come from **how it is built**, not from **checking it
afterward**. "Correct by construction" vs. "correct by validation."

---

## STATUS (working draft)

- [x] Running examples + JSON Pointer primer
- [x] Point 1 — When a `NewType` (brand) is the right tool (1a invariant test,
      1b vs kw-only, 1c who produces + preventing bad wraps)
- [x] Point 2 — Correct by construction / airtight construction (broken
      SafeHtml, Level A/B, immutability, closure, payoff, total-vs-partial,
      naming convention)
- [~] Point 3 — The decision tree (sketched below; wants prose)
- [ ] Static vs runtime enforcement (types evaporate at untyped boundaries) --
      partly embedded in Pt 1c / Pt 2; wants its own section
- [ ] Recoverable vs unrecoverable invariants -- the axis behind Level A/B;
      anchor on the JSON Pointer "valid-but-wrong" explanation
- [ ] Language reality table (Rust/Haskell/OCaml/F#/Go/Python/TypeScript)
- [ ] When to use which / proportionality (don't over-build; false confidence)
- [ ] The provenance family (SafeHtml/SqlFragment, taint, units, normalization)

Verified with mypy / runtime during design (snippets below are checked).

---

## Running examples

Three examples recur:

- **A pair of ID types** (`UserId` / `ProductId`) — a type with *no invariant*
  (every `int` is valid). Illustrates roles/brands (Point 1).
- **JSON Pointer** — a type with an *unrecoverable* invariant (escaping).
  Illustrates why some guarantees can only be made at construction (Points 2, 3).
- **`SafeHtml`** — a type whose invariant is real but must sometimes accept a
  pre-made value. Illustrates the "loud, auditable escape hatch" (Point 2).

### The running example: JSON Pointer

A JSON Pointer ([RFC 6901](https://www.rfc-editor.org/rfc/rfc6901)) is a string
that names a location inside a JSON document — a sequence of **reference
tokens**, each introduced by `/`:

- `/axes/x` -> "member `axes`, then member `x`"; `/values/0` -> "member
  `values`, then array index `0`".
- The empty string `""` names the whole document.

Because `/` separates tokens, any `/` *inside* a token's content is escaped to
`~1`, and any literal `~` to `~0`. So an object key `"x/y"` becomes the token
`x~1y`, and the pointer to it is `/axes/x~1y`.

(You'll also see a `#`-prefixed form, `#/axes/x~1y` — the same pointer written as
a URI fragment. We use the plain form here.)

That escaping rule — `/`->`~1`, `~`->`~0` — is the invariant the examples care
about. Full spec: RFC 6901 (terse; the primer above is enough for this doc).

---

## Point 1 — When a `NewType` (brand) is the right tool

### 1a. The deciding question: *is there an invariant?*

> Is every value of the base type a valid member of your type, or only some of
> them?

- Every base value valid -> **no invariant** -> use `NewType`.
- Only some valid -> **there is an invariant** -> `NewType` is the wrong tool
  (it cannot enforce "some").

The canonical correct use has no validity rule — you only want the checker to
stop you confusing two things that share an underlying type:

```python
UserId    = NewType("UserId", int)
ProductId = NewType("ProductId", int)

def charge(user: UserId, product: ProductId) -> None: ...

u, p = UserId(5), ProductId(99)
charge(u, p)      # ok
charge(p, u)      # type error: arguments swapped  <- the only bug NewType prevents
```

"Is `5` a *valid* `UserId`?" is meaningless — every int is one. So there is
nothing to check, and `UserId(5)` accepting any int is *correct and desirable*.
The only failure mode is a *mix-up* (a type-checking concern), not a *validity*
violation. `NewType` is a zero-cost, type-check-only role label: at runtime
`UserId(5)` *is* `5`.

**The crux:** the same behavior — a brand's constructor accepting any base value
— is a **feature** for `UserId` (constructing from an int is legitimate) and a
**catastrophe** for `Pointer` (constructing from a raw string bypasses
escaping). The discriminator is solely *whether an invariant exists*.

Caveat: even for `UserId`, `NewType` won't stop a *deliberate*
`UserId(some_product_int)` — that's a cast. It prevents *accidental* confusion,
which is its whole job.

Rule of thumb: **`NewType` distinguishes roles/identities; it does not enforce
validity.**

### 1b. `NewType` vs. keyword-only parameters

> kw-only is a property of a *function signature*, enforced *at one call site*,
> relying on a human to read the names. `NewType` is a property of the *value*,
> enforced *everywhere the value flows*, checked mechanically.

What mypy flags (verified):

| Failure | kw-only alone | `NewType` |
|---|---|---|
| Positional swap `charge(p, u)` | caught (forbids positional) | caught (type mismatch) |
| Right name, **wrong value** `charge(user=p, product=u)` | MISSED | caught |
| Wrong role in **assignment** `uid: UserId = p` | N/A | caught |
| Wrong role in **collection/return** `ids.append(p)` | N/A | caught |
| Two args of the **same** role `transfer(src=b, dst=a)` | caught (names) | MISSED (both UserId) |

Why "just use kw-only" doesn't cover it:
1. kw-only stops *positional* swaps, not *wrong values* — it never checks the
   value under `user=` is actually a `UserId`. It converts "swapped by position"
   into "must be named," then trusts the human to name correctly.
2. kw-only is call-site-only — silent on assignments, returns, dict keys, list
   elements. `NewType` guards the value anywhere it travels.

Why `NewType` isn't a full answer either: two args of the *same* role
(`transfer(src, dst)`, both `UserId`) — `NewType` can't disambiguate; names are
the only guard. That's where kw-only is the right and only tool.

They **compose**: different-typed values -> often both (`NewType` for value
safety, kw-only for readability); same-typed args -> kw-only only; values that
flow through assignments/returns/collections -> `NewType`.

### 1c. Who produces `NewType` values, and preventing bad wraps

"Prevent wrapping a bad value" forks on the invariant question:

- **No invariant (correct use):** there *are* no bad values — only *mislabels*
  (`UserId(a_product_id)`), which you cannot mechanically prevent. Organizational
  problem.
- **Invariant present:** `NewType` structurally *cannot* prevent a bad wrap —
  and that inability *is the signal* to use a smart constructor instead.

The "I wrapped it to satisfy the type error" anti-pattern = a base-typed value
where a branded one is expected. Two cases, only a human can tell them apart:
- (a) right role, not branded yet -> fix by branding **upstream at its source**;
- (b) wrong role -> the wrap is a **lie suppressing a real bug**.

**Who produces them — branding is a boundary/authority responsibility.** A brand
is a **vouching** ("I certify this int is a user id"). The producer is whoever
can honestly vouch — the layer where raw data first becomes meaningful:

```python
# boundary (imperative shell): the AUTHORITY mints
def load_user(row: dict) -> User:
    return User(id=UserId(row["id"]), ...)   # DB is the authority for user ids
def user_id_param(raw: str) -> UserId:
    return UserId(int(raw))                   # request-parse boundary vouches

# interior (functional core): only CONSUMES
def greet(uid: UserId) -> str: ...            # takes a UserId; never writes UserId(...)
```

Interior code minting a brand = *forging provenance* for something it doesn't
know = the bug itself. Same edge as "parse, don't validate."

Making bad wraps much less likely (Python can't fully prevent them):
1. **Remove the temptation** — interior functions *take* branded types, so no
   one is holding a bare value that "needs" a wrap.
2. **Concentrate the mints** at a few boundary sites / one factory module, so
   there's a single grep/lint target ("no `UserId(` outside boundary"). Turns
   *silent wrap anywhere* into *auditable wrap at known sites*.
3. **Accept convention here** — with no invariant, the worst outcome is a
   mislabel (logic bug), not corruption; convention + review is proportionate.

The tell you've outgrown `NewType`: the moment you want to *validate at the mint*
(reject bad values), you've admitted an invariant -> promote to a value object.

---

## Point 2 — Correct by construction (airtight construction)

A class is *not* automatically safe. The broken version:

```python
class SafeHtml:
    def __init__(self, s: str) -> None:
        self._html = s          # accepts ANY string -- "safe" in name only
SafeHtml("<script>")            # no safer than a bare str; same leak as NewType
```

The point is not "make it a class" — it's **"the raw form cannot come in the
front door."**

The fix depends on one domain question:

> Do you ever legitimately need to accept an already-valid value you *cannot
> rebuild from ingredients*?

**Level A — No -> eliminate the raw path** (most airtight). JSON Pointer:
rebuild from tokens, so no formed-pointer input exists:

```python
def pointer(*tokens: str) -> str:            # only ingredients; escaping internal
    ...
# pointer("axes", "x/y") -> "#/axes/x~1y"    even misuse yields a VALID pointer:
# pointer("a/b")         -> "#/a~1b"         one escaped token, never malformed
```

There is no public way to express a bad value.

**Level B — Yes -> keep the safe path default, make the unsafe path explicit and
named.** `SafeHtml` must accept already-safe HTML (a template's output; can't
re-escape without double-escaping `&amp;`->`&amp;amp;`):

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

Safer even though `assume_safe` still lets an unescaped string in: the safe path
is the default, `SafeHtml("<script>")` now raises, and the unsafe path is *named
as an assertion* -> never accidental, and `grep assume_safe` audits every trust
decision.

**"Airtight" is not binary — it's how loud and auditable the unsafe path is:**
- **Level A** (Pointer): unsafe path doesn't exist -> rebuild from ingredients.
- **Level B** (SafeHtml): unsafe path must exist -> name it, make it greppable.
- **Brand / broken-SafeHtml**: unsafe path is the default front door, silent.

### Why JSON Pointer is the textbook "unrecoverable invariant" (anchors Pt 3)

The invariant is "every `/` or `~` in a token's content was escaped." Violating
it produces a *valid* pointer, not a malformed one:
- Correct: tokens `["axes","x/y"]` -> escape -> `/axes/x~1y`.
- Buggy: same intended key `"x/y"`, forgotten escape -> `/axes/x/y`.

`/axes/x/y` in isolation is a perfectly valid pointer meaning *three* tokens
`["axes","x","y"]`. You cannot tell whether the producer intended three tokens
(correct) or two and forgot to escape (bug). Both intents map to valid strings;
the flat string doesn't record which. So no validator can flag it -- nothing is
malformed. That's "unrecoverable": the only fix is to never emit an unescaped
token -> build from tokens (Level A), where escaping is applied by construction
and the intent-corrupting `/axes/x/y` is *unreachable* through the API.

### Airtight = controlled construction AND immutability

Construction control is necessary but not sufficient. `__slots__` restricts
*which* attributes exist; it does NOT make them read-only (verified: reassigning
a slot works; `@dataclass(frozen=True)` raises `FrozenInstanceError`):

```python
s = SafeHtml.escape("safe")
s._html = "<script>"          # __slots__ does NOT stop this -- invariant broken after construction
```

Close it with: store immutable underlying data (`str`, `tuple`); expose no
mutators and never leak an internal mutable reference; make the wrapper's fields
read-only via frozen semantics (frozen dataclass / frozen msgspec Struct /
custom `__setattr__`). **Airtight = controlled construction *and* immutability.**

### The invariant must survive every operation (closure)

The most common hole is a convenience method, not the constructor:

```python
class Pointer:
    def with_suffix(self, s: str) -> str:      # LEAK: raw str out, invariant gone downstream
        return str(self) + s
    def __truediv__(self, token: str) -> "Pointer":  # CLOSED: returns Pointer, escapes token
        ...
```

Every operation must be **closed over the refined type** — take and return the
refined type, re-establishing the invariant internally (`SafeHtml + SafeHtml ->
SafeHtml`, `Pointer / token -> Pointer`). A method returning the raw base type
undoes the constructor's work.

### Smaller details

- **The payoff:** once airtight, downstream code never re-checks — the type *is*
  the proof. Pay once at the boundary; every consumer gets the guarantee free
  ("parse, don't validate").
- **Total vs partial constructors:** `pointer(*tokens)` is *total* (every input
  -> some valid value, even misuse -> valid-but-wrong), so no failure path. A
  validating `Percentage(int)` is *partial* (can reject). Prefer total when the
  domain allows; use partial to reject invalid input at the boundary (a
  "parser" is a partial constructor).
- **Naming convention:** `assume_safe` / `_unchecked` / `unsafe_` is established
  — Rust's `str::from_utf8_unchecked`, `get_unchecked`, the `unsafe` keyword.
  Safe API default; bypass loudly labeled.

---

## Point 3 — The decision tree (TO EXPAND)

```
1. Are only SOME base values valid? (Is there an invariant?)
   NO  -> NewType / brand.  (role label; nothing to enforce)  -- done
   YES -> 2.

2. Can you check the invariant on a FINISHED value? (recoverable?)
   YES (decidable)     -> value object that VALIDATES in its constructor
                          (accepts the finished form, raises on bad input;
                          e.g. Percentage: 0 <= v <= 100).
   NO  (unrecoverable) -> construct-ONLY smart constructor: take INGREDIENTS,
                          never the finished form (e.g. Pointer from tokens).

3. Can values arrive from untyped / JSON / public callers?
   YES -> ALSO add a runtime guard; static types don't fire at those boundaries.

4. Must you sometimes accept an already-valid value you CANNOT rebuild?
   NO  -> omit the raw path entirely.               (Level A -- Pointer)
   YES -> explicit, NAMED unsafe hatch, safe default (Level B -- assume_safe)
```

---

## TO FOLD IN (pending sections)

- **Static vs runtime enforcement.** A type guards type-checked call sites only;
  it evaporates at untyped/deserialization/public boundaries. (msgspec verified:
  validates on decode, not on direct construction; `NewType` has no runtime
  identity at all.) So: types for interior call sites; a runtime guard at
  boundaries.
- **Recoverable vs unrecoverable invariants.** The axis behind Level A/B and
  step 2 of the tree; anchor on the JSON Pointer "valid-but-wrong" explanation
  above.
- **Language reality table.** Rust/Haskell/OCaml/F#: hide the constructor, export
  a smart constructor -> compiler-guaranteed airtight, zero cost. Go: unexported
  fields + package constructor -> airtight across package boundaries. TS: branded
  types are compile-only; a `#private` field buys runtime identity. Python: no
  true privacy -> ceiling is "no public path accepts a raw value + runtime guard
  + clearly-private seam."
- **Proportionality / when to use which.** Match effort to blast radius,
  producer breadth, whether callers are type-checked, and recoverability. The
  trap: a guard that *looks* stronger than it is (a brand on a public surface) ->
  false confidence, worse than none.
- **The provenance family.** Same shape everywhere — the type carries a fact the
  bytes don't: sanitization (`SafeHtml`/`SqlFragment`; why HTML-escapers, SQL
  tagged templates, and Python 3.14 t-strings exist), taint (`Untrusted[str]` vs
  `Trusted[str]`), units (`Meters` vs `Feet`), normalization (`LowercasedEmail`,
  `AbsolutePath`, `NfcString`).
