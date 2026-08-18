# int-llm — Remediation plan: explicit state, and the defects the gate cannot see

| Field | Value |
|---|---|
| Document | `docs/concepts/RNG_STATE_CARRIED_UPGRADE_PLAN.md` |
| Class | **Plan (non-implementation).** It authorises no code change |
| Author | Straightedge (Claude Opus 5), 2026-08-18 |
| Repository | `kryptosmatrix/int-llm` (fork). Upstream is `nmicic/int-llm` |
| Read at | `0b4b6d04eb3e9e969d125804a154309eed3be9de`, branch `main`, clean tree. **Every line reference below was read there.** Nothing was compiled or run |
| Covers | Six defects across `fp_math.h` and `llama_int.c`, and the sequencing method that keeps the determinism gate meaningful while they are fixed |

---

## 1. The method, because it decides the order of everything else

`make determinism` compares an FNV-1a-64 hash — currently `c0d933ea340452ec` — over the raw
outputs of `fp_determinism.c`'s twelve-function, 4,352-point grid. It is an excellent gate and it
is easy to over-trust, because **it adjudicates three kinds of change completely differently.**

**Class A — the change must not move the hash.** Relocating the generator's state is the example:
`xorshift64` is a pure function of its state, so moving where that state lives changes no
arithmetic. Here the hash is a true oracle. Green means correct; red means you broke it, with no
third reading. These can be batched relatively freely, because a failure bisects cleanly.

**Class B — the change will move the hash.** Replacing runtime-computed CORDIC tables with
precomputed constants is the example: sine and cosine are in the grid, so the hash moves whether
you did it right or wrong. **Here the hash is not a gate, only a change detector.** The real proof
is separate — compare the precomputed values against the runtime-computed ones on every target
platform — and only then is the golden deliberately re-blessed. **Class B changes land strictly one
at a time**, because a re-blessed golden is trustworthy only if exactly one intended change moved
it. Two together and you have blessed a hash you cannot fully account for, and every later run
inherits that.

**Class C — the hash cannot see the change at all**, and this is the dangerous one. The gate
covers `fp_math.h` only. `make regression` runs the 335 unit tests in `fp_test.c` plus both
determinism binaries; `make llama` is a manual command requiring a model directory. **No automated
gate touches `llama_int.c`.** Three of the six defects below live there. A green gate says nothing
about them — it will stay green whether they are present, fixed, or made worse — so every Class C
change needs a purpose-built test written *before* the fix, and the fix is only proven when that
new test fails on the old code and passes on the new.

**Sequencing follows from the classes, not from taste.** Do all Class A work first, while the gate
is still an oracle. Then Class C, each with its own new falsifier, since it is invisible to the
gate either way. Then Class B, singly, each with its own platform proof and its own deliberate
re-bless. **And run the whole gate before touching anything**, recording the pass, so a later
failure cannot be blamed on the starting state.

---

## 2. Defect register

### D1 — The generator's state is a file-static global · **Class A**

`fp_math.h:927` declares `static unsigned long long fp_rng_state = 42;`, mutated by `fp_rng_next`
(929–934) and consumed by `fp_rng_uniform` (937–940), `fp_gaussian` (944–) and `fp_shuffle_ints`
(955–961).

**The requirement for explicit state already exists and is met by reaching into that global from
another file.** `microgpt_int.c:855` reads `fp_rng_state` to write it into a checkpoint; `:936`
and `:1016` assign it back on restore. A training run that resumes must resume its stream — a real
need, served by the most fragile mechanism available.

**The header documents the hazard instead of fixing it** (118–128): `fp_math_init()` is "not
thread-safe"; "all other functions are stateless EXCEPT `fp_rng_*` which share global state";
multi-threaded callers should "give each thread its own `fp_rng_state`", which the API gives them
no way to do; and because everything is `static inline`, **each translation unit gets its own
copy**, so callers must not "expect RNG streams … to be shared across TUs".

**Severity, stated accurately:** each program here is effectively one RNG consumer, so the per-TU
copies do not currently collide. This is a **latent hazard, not a live bug**. It becomes live the
moment two translation units in one program share a stream, or two threads do, and the checkpoint
round-trip is where it surfaces first because it assumes the state it saves is the state the
drawing code uses.

**Fix — §3 specifies it in full.** An explicit state type and a reentrant `_r` API, with every
existing name kept as a wrapper over one default instance.

**Proof:** the golden must not move. The generator is inside it — `fp_determinism.c:202` seeds
`0x123456789ABCDEF` and `:204-205` mix `fp_rng_next` and `fp_rng_uniform` into the hash — so a
refactor that perturbs the stream by a single draw fails loudly. Plus the new falsifier in §3.

### D2 — `xorshift64` has an absorbing state at zero and nothing validates the seed · **Class A**

If the state is ever `0`, each of the three shift-and-exclusive-or steps leaves it `0`, and the
generator returns `0` for every subsequent call. No code path checks. Seeds are assigned directly
by `fp_test.c` (nine sites), `fp_determinism.c:202`, and — the one that matters —
`microgpt_int.c:936` and `:1016`, which restore **whatever the checkpoint file contains**. A
corrupt or truncated checkpoint carrying zero **silently turns randomness off** rather than
failing, and a training run continues with a dead generator.

**Fix:** the seed entry point guards it. Two options and they are not equal. Rejecting is honest
but gives the function a failure mode in a header with no error convention, which then propagates
into `fp_gaussian`'s callers. Normalising substitutes a fixed non-zero constant and is total.
**Recommendation: normalise, and document the substitution in the header.** The decision is owed
before the fix lands.

**Proof:** hash-preserving in practice, because no seed currently in use is zero — 42,
`0x123456789ABCDEF`, 123, 456, 789, 999 and 1000. A new unit test seeds zero and asserts the
generator still produces a non-constant sequence; on the old code that test fails by returning
zeros forever.

### D3 — `fp_math_init` computes constants into mutable globals at runtime · **Class B**

`fp_math_init` (833–858) lazily fills `FP_PI` via Machin's formula, a 48-entry `fp_cordic_angles`
table via `arctan(2^-i)`, and `fp_cordic_gain` via a product **and a call to `fp_sqrt`**, guarded
by an `fp_math_initialized` flag. `fp_sincos` (861–902) calls it implicitly on entry. The header
calls it idempotent but **not thread-safe**, and each translation unit gets its own copies.

**Fix:** precompute the table and the two constants and carry them as `const` data, so the header
has no lazy initialisation and no mutable state; or, if runtime computation must stay, make
initialisation explicit, once, and callable by the consumer rather than implicit inside `sincos`.

**Why this is Class B and must land alone.** Sine and cosine are in the golden grid, so the hash
moves. Worse, it may move *legitimately*: the header's own notes say the 32-bit path can differ by
architecture and compiler — which is why `determinism-portable` exists as a separate target — so
precomputed constants might disagree with runtime-computed ones on some platform and be **more**
correct, not less. **The proof is therefore not the hash.** It is a comparison harness that
computes both ways on each target and reports any difference before a single constant is frozen.
Only when that is clean is the golden deliberately re-blessed, in a commit that does nothing else.

### D4 — The matmul accumulator narrows by a bare cast · **Class C**

`llama_int.c:1033-1036` writes `out[i + 0] = (fixed_t)(acc0 >> FP_PRECISION);` and three more like
it in the four-way unrolled path, with `:1044` doing the same in the remainder loop. The
accumulator is `int128_t`; `fixed_t` is 64-bit. **The cast is unchecked**, so a result whose true
value exceeds the 64-bit range is silently truncated to its low bits — a wrong answer that looks
like a right one, with no diagnostic anywhere.

**Fix:** a checked narrow that detects the out-of-range case. What it should *do* then is a design
decision for this project rather than something to copy: refuse, saturate, or record and continue.
Whichever is chosen must be the same at all five sites.

**Proof, and it must be written first:** no automated gate covers this file, so construct a matrix
and vector whose true product exceeds the 64-bit range, assert the current code truncates, then
assert the fixed code does the chosen thing instead. Without that test the change is unverifiable
and the green gate is meaningless.

### D5 — `rmsnorm` adds epsilon with an unchecked 64-bit add · **Class C**

`llama_int.c:1052` computes `fixed_t mean_sq = (fixed_t)((sum_sq >> FP_PRECISION) / dim);` — the
shift and divide happen correctly inside 128 bits and the narrowing is last, which is the right
order — but the narrowing is again **a bare cast**. Then `:1053-1054` declare
`fixed_t eps = 2814749767LL` and do `mean_sq += eps` as a plain 64-bit add. A value that sits
inside `Int64` but within `eps` of its maximum **overflows**, and under `-fwrapv` it wraps
silently and the wrapped value feeds `fp_inv_sqrt`.

**Fix:** a checked add, and the same checked narrow as D4 one line above it. The narrowing is the
first-order hazard and the addition the second.

**Proof:** a directed unit test at the boundary — a vector whose sum of squares lands within `eps`
of the ceiling — asserting the wrap on old code and the chosen behaviour on new.

### D6 — The softmax accumulator is 64-bit, and its guard is reachable only by wrapping · **Class C**

`llama_int.c` declares `fixed_t sum = 0`, accumulates `fp_safe_exp` results across the row, and
then guards with `if (sum > 0)`, leaving the row **un-normalised** when the guard is false.

For rows up to 32,767 terms the guard is dead: after max-subtraction the largest term is
`exp(0) = FP_ONE` exactly, so the sum is at least one. Above that ceiling the 64-bit accumulator
can wrap under `-fwrapv` — to exactly `−2^63` at a uniform 32,768 terms, and to exactly `0` at
65,536 — at which point the guard genuinely fires and the function silently returns an
un-normalised row. **Modern context lengths exceed that ceiling**, which is why the ARCHE-Int
oracle widened its own accumulator to 128-bit and declared the divergence.

**Fix:** widen the accumulator to `int128_t`, which makes the bound `n < 2^79` and unreachable.
The guard then becomes a genuine defensive invariant rather than a live path.

**Proof:** a row of 40,960 terms whose expected values come from an independent exact-rational
computation, **not** from this implementation. On the old code it wraps; on the new it does not.

**Note the interaction with D4.** Widening this accumulator changes results for rows above the
ceiling — which is the point — so it is not silent. It stays Class C because no automated gate
covers the file either way.

---

## 3. The explicit-state design, in full

The C standard library solved this twice — `rand`/`rand_r`, `strtok`/`strtok_r` — and that pattern
is what makes the change adoptable rather than disruptive.

**Add** a state type wrapping the `unsigned long long`; reentrant functions taking it by pointer
(`fp_rng_next_r`, `fp_rng_uniform_r`, `fp_gaussian_r`, `fp_shuffle_ints_r`); `fp_rng_seed(state,
seed)` as the **only** supported way to set a stream, carrying D2's guard; and an accessor pair so
serialisation never touches the variable directly.

**Keep** every existing name as a thin wrapper over one default instance — `fp_rng_next()` becomes
`fp_rng_next_r(&fp_rng_default)`. Every current caller compiles unchanged and produces identical
bytes. The only callers that must change are the thirteen assigning `fp_rng_state` directly: nine
in `fp_test.c` (646, 651, 656, 665, 674, 683, 694, 951, 1032), one in `fp_determinism.c` (202),
three in `microgpt_int.c` (855, 936, 1016).

**Add one falsifier the existing suite cannot express**, because everything currently there would
pass a wrong implementation: two independently seeded states drawing **interleaved** must produce
the same sequences they produce in isolation. A `_r` API that secretly shared state passes every
other test in the repository.

**Optionally, last:** a compile-time switch that removes the default instance entirely, so a
consumer wanting no global state fails to link rather than silently getting one. Nothing here
needs it; a downstream deterministic engine does, and it turns "we don't use the global" from a
claim into a build failure.

---

## 4. Order of work

Run the full gate first and record the pass. Then: **D1** and **D2** together (Class A, one gate
run proves both). Then **D4**, **D5**, **D6** in that order, each with its falsifier written
before its fix, since the gate is blind to all three and D5's narrowing shares D4's checked-narrow
helper. Then **D3** alone, with its platform comparison harness, and a deliberate golden re-bless
in a commit that does nothing else.

**Upstream.** The wrapper-over-default design is what makes D1 and D2 offerable to `nmicic/int-llm`
rather than fork-only divergence: no caller breaks, no byte moves, and one command proves both
claims. Offer those two first and alone. D4 through D6 change numerical behaviour and deserve
their own conversation, since they may be deliberate trade-offs rather than oversights — the
matmul cast in particular is fast, and its author may have accepted the truncation knowingly.

---

## 5. Risks

**The gate is a hash, so it reports *that* something moved, not *what*.** When a Class A phase
fails, bisect using `fp_determinism.c`'s per-function sub-hashes rather than the combined value.

**Class C work has no safety net by construction.** Three defects sit in a file no automated target
touches. The discipline that replaces the missing net is writing the falsifier first and watching
it fail on the current code; a test written after the fix proves only that the fix agrees with
itself.

**D3 is the one that can quietly bless a wrong constant.** If the comparison harness is skipped and
the golden is re-blessed because "the change was intended", any platform disagreement is frozen
into the reference from then on.

**Nothing here was compiled or executed.** Line numbers, seeds, constants and the golden value were
read from the working tree; `make` was not invoked. Treat every claim as re-checkable and re-check
the ones you rely on.

---

*Straightedge (Claude Opus 5), 2026-08-18. Plan only — it authorises no code change. Two decisions
are owed before work starts: D2's reject-or-normalise, and D4's refuse-saturate-or-record.*
