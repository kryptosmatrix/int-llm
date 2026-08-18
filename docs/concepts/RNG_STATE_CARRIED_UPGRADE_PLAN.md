# int-llm — Plan: carry the RNG state explicitly

| Field | Value |
|---|---|
| Document | `docs/concepts/RNG_STATE_CARRIED_UPGRADE_PLAN.md` |
| Class | **Plan (non-implementation).** It authorises no code change |
| Author | Straightedge (Claude Opus 5), 2026-08-18 |
| Repository | `kryptosmatrix/int-llm` (fork). Upstream is `nmicic/int-llm` |
| Read at | `0b4b6d04eb3e9e969d125804a154309eed3be9de`, branch `main`, clean tree. **Every line reference below was read at that commit** |
| Goal | Make the generator's state an explicit value the caller owns, without changing a single output byte |
| Non-goal | Changing the algorithm. `xorshift64` stays exactly as it is |

---

## 1. Why

`fp_math.h:927` declares `static unsigned long long fp_rng_state = 42;` — a file-static global
mutated by `fp_rng_next` (929–934). Everything downstream inherits it: `fp_rng_uniform`
(937–940), `fp_gaussian` (944–), and `fp_shuffle_ints` (955–961).

**The codebase already needs explicit state and currently gets it by reaching into that global
from another file.** `microgpt_int.c:855` reads `fp_rng_state` to write it into a checkpoint,
and `:936` and `:1016` assign it back on restore. That is a real requirement — a training run
that resumes must resume its random stream — met by the most fragile mechanism available.

**The header already documents the hazard rather than fixing it.** Lines 118–128 state that
`fp_math_init()` is not thread-safe, that "all other functions are stateless EXCEPT `fp_rng_*`
which share global state", that multi-threaded callers should "give each thread its own
`fp_rng_state`" — which the API provides no way to do — and that because everything is
`static inline`, **each translation unit gets its own copy**, so callers should not "expect RNG
streams … to be shared across TUs".

Be precise about the severity, because overstating it would weaken the case. Today each program
here is effectively one consumer of the RNG, so the per-TU copies do not currently collide. **It
is a latent hazard, not a live bug** — and it becomes a live one the moment two translation units
in one program both draw from the stream and expect it to be shared, or two threads do. The
checkpoint round-trip in `microgpt_int.c` is the place it would surface first, because it assumes
the state it saves is the state the drawing code uses.

**One outright defect, found while reading.** `xorshift64` has an absorbing state at zero: if
`fp_rng_state` is ever `0`, every shift-and-xor leaves it `0` and the generator returns `0`
forever. Nothing validates the seed. `fp_test.c` and `fp_determinism.c` assign seeds directly,
and `microgpt_int.c:936` restores whatever a checkpoint file contains — **a corrupt or truncated
checkpoint carrying zero silently turns the generator off** rather than failing. §3 phase 1 fixes
this as part of the seed entry point.

---

## 2. The design: `_r` alongside, not instead

The C standard library has solved this twice — `rand`/`rand_r`, `strtok`/`strtok_r` — and the
pattern is what makes the change adoptable rather than disruptive.

Introduce a state type and a reentrant API taking it by pointer: a small struct wrapping the
`unsigned long long`, plus `fp_rng_next_r`, `fp_rng_uniform_r`, `fp_gaussian_r` and
`fp_shuffle_ints_r`. Add `fp_rng_seed(state, seed)` as the **only** supported way to set a
stream, and an accessor pair for serialisation so a checkpoint never touches the variable itself.

**Keep every existing name as a thin wrapper over one default instance.** `fp_rng_next()` becomes
`fp_rng_next_r(&fp_rng_default)`. Every current caller compiles unchanged and produces identical
bytes; the only callers that must change are the ones assigning `fp_rng_state` directly, and they
change to `fp_rng_seed`.

**The sequence cannot move, and that is the whole safety argument.** `xorshift64` is a pure
function of its state — three shifts and three exclusive-ors. Relocating that state from a
file-static to a struct field changes no arithmetic. Anything that alters an output byte is a
mistake in the refactor, not a consequence of it, and §4's gate is designed to catch exactly that.

---

## 3. Phases

Each phase ends green on the same gate (§4), so any phase can be the last one merged.

**Phase 1 — add the API, change no behaviour.** Introduce the state type, the `_r` functions, the
default instance, `fp_rng_seed` with its **zero-seed guard**, and the accessor pair. Existing
names become wrappers. No caller edits. This is the phase that must prove the golden hash is
untouched, because nothing else in it is observable.

*Zero-seed decision owed:* `fp_rng_seed` should either reject zero or normalise it to a fixed
non-zero constant. Rejecting is honest but gives the function a failure mode in a header with no
error convention; normalising is silent but total. **Recommendation: normalise, and document the
substitution in the header** — the alternative puts an error path into `fp_gaussian`'s callers
for a case that only arises from corrupt input.

**Phase 2 — migrate the direct assignments.** Thirteen sites: nine in `fp_test.c` (646, 651, 656,
665, 674, 683, 694, 951, 1032), one in `fp_determinism.c` (202), three in `microgpt_int.c` (855
read, 936 and 1016 restore). All become `fp_rng_seed` or the accessors. Mechanical, and the gate
proves it: the golden and the regression suite must be byte-identical before and after.

**Phase 3 — give the checkpoint path its own state.** `microgpt_int.c`'s save/restore stops
touching a global and carries a generator alongside the model state it belongs to. This is the
phase that actually removes the latent hazard rather than tidying around it, and it is where the
zero-seed guard earns its place, since a restored checkpoint is the one path that can supply one.

**Phase 4, optional — prove the absence of hidden state.** A compile-time switch that removes the
default instance entirely, so a consumer that wants no global state can fail to link rather than
silently get one. Nothing in this repository needs it; a downstream deterministic engine does, and
it converts "we don't use the global" from a claim into a build failure.

---

## 4. The gate, and it is one command per phase

`make determinism` must reproduce `tests/determinism_golden.txt` — currently `c0d933ea340452ec`,
an FNV-1a-64 over the raw outputs of the twelve-function, 4,352-point grid. **The generator is
inside that hash**: `fp_determinism.c:202` seeds `0x123456789ABCDEF` and `:204-205` mix
`fp_rng_next` and `fp_rng_uniform` into it. So a refactor that perturbs the stream by one draw
changes the hash and fails loudly.

Also required each phase: `make determinism-portable`, which guards the 32-bit path the header's
own notes say can differ by architecture and compiler; `make regression`; and `make test`, whose
RNG section (`fp_test.c:646-700`) already asserts sequence determinism, uniform range, and the
Gaussian's mean and spread.

**One falsifier the refactor must add**, because the existing suite would pass a wrong version of
it: two independently seeded states drawing interleaved must produce the same sequences they
produce in isolation. Nothing today can express that test, which is precisely the property the
change exists to create — and a `_r` API that secretly shares state would pass every test above.

---

## 5. Scope boundary

**The CORDIC initialisation globals are the same class of defect and are deliberately not in this
plan.** `fp_math_init` (833–858) lazily fills `FP_PI`, a 48-entry angle table and a gain constant
into mutable statics, guarded by a flag, and the header calls it idempotent but not thread-safe.
That is real and worth fixing, but the fix is different in kind — precompute the tables as
constants, or make initialisation explicit and once — and folding it in would double the change
and its risk. **Name it, plan it separately.**

---

## 6. Offering it upstream

The wrapper-over-default design is what makes this acceptable to `nmicic/int-llm` rather than a
fork-only divergence: no existing caller breaks, no output byte moves, and the golden hash proves
both claims in one command. If it is offered upstream, offer **phase 1 alone** first — it is the
whole API with none of the migration — and let the caller migration follow separately.

---

## 7. Risks

**The gate is a hash, so it tells you *that* something moved and not *what*.** If a phase fails
the golden, bisect by function: `fp_determinism.c` computes per-function sub-hashes as well as the
combined one, and the per-function values are what localise a break.

**Phase 2 is mechanical and therefore the easiest to do carelessly.** Thirteen sites, several
inside benchmark loops where a mistyped seed changes results without changing correctness.

**Nothing here was compiled or run in producing this plan.** Line numbers and the golden value were
read from the working tree at the pinned commit; `make` was not invoked. The first action of
whoever takes phase 1 should be to run the §4 gate **before** changing anything, and record that it
passes, so that a later failure is attributable to the change rather than to the starting state.

---

*Straightedge (Claude Opus 5), 2026-08-18. Plan only — it authorises no code change, and the
zero-seed decision in §3 phase 1 is owed before that phase begins.*
