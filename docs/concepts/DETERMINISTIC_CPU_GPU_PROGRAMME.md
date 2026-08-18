# int-llm — Programme plan: a fully deterministic integer LLM on CPU, GPU, or both

| Field | Value |
|---|---|
| Document | `docs/concepts/DETERMINISTIC_CPU_GPU_PROGRAMME.md` |
| Class | **Programme plan (non-implementation).** Authorises no code change |
| Author | Straightedge (Claude Opus 5), 2026-08-18 |
| Repository | `kryptosmatrix/int-llm` (fork); upstream `nmicic/int-llm` |
| Goal, from Ash | Correct, update and enhance the design into a **completely deterministic LLM running on CPU, GPU, or both together, on Apple Silicon** |
| Ruling 1 | **The GPU and determinism architecture INHERITS ARCHE-Int's specification** rather than deriving its own. One specification, two implementations, in different languages |
| Ruling 2 | The determinism golden **may move**. Behaviour changes move it and that is expected. §3 states the one discipline that survives that |
| Read at | `int-llm` `0b4b6d0` (clean, branch `main`); ARCHE at `plumbline/arche-int-plans` head; ANANKE `README.md` at working head. **Nothing was compiled or run** |
| Companion | `RNG_STATE_CARRIED_UPGRADE_PLAN.md` — the six-defect register this programme's stage 1 and 2 execute |

---

## 1. What "deterministic" has to mean here, stated before anything is built

The word carries four separable claims, and they need different proofs. Conflating them is how a
project ends up believing it has all four because it tested one.

**R1 — run repeatability.** Same binary, same inputs, same machine, same process: identical bits.
Already true here, and the weakest claim.

**R2 — process independence.** A fresh process reproduces the previous one exactly. This is where
hidden mutable state bites, which is why the six-defect plan's D1 and D2 are prerequisites rather
than tidying.

**R3 — partition invariance.** The result does not depend on *how the work was divided* — thread
count, shard count, threadgroup size, serial versus parallel, or the CPU/GPU split. **This is the
claim "CPU, GPU, or both together" actually rests on**, and it is the only one that constrains the
arithmetic itself rather than the plumbing.

**R4 — platform invariance.** Same inputs, different machine or architecture: identical bits.
`make determinism-portable` reaches at this for the 32-bit backend. **The current build cannot
honestly claim it** — see D7 below.

A programme that does not say which of the four it is claiming, per stage, will drift into
claiming all of them. Every stage in §4 names its own.

---

## 2. Three repositories already touch this, and the overlap is the strategic fact

**ARCHE-Int** (twelve blueprints under `ARCHE/docs/blueprint`) specifies exact integer inference
for Apple Metal: the Q16.48 numeric ABI with its seventeen-rule operator table, the two-limb
`W128` accumulator rendered in full for Metal Shading Language, the three conditions under which a
GPU partition stays bit-identical, threadgroup-invariance tests, and an ordered per-operator digest
ladder that localises a divergence to one operator in one layer. **It has never been executed** —
no build, no test, across six judging rounds and roughly half a megabyte of specification, and it
currently stands at `BLUEPRINT_JUDGE=FAIL` with implementation blocked.

**ANANKE** (`kryptosmatrix/ANANKE`) is a *working* deterministic CPU/GPU pipeline engine in Swift.
Its stated core guarantee is the same pipeline over the same inputs on CPU or GPU, with one worker
or eight, parallel or serial, producing an identical SHA-256 receipt digest. It ships
`AnankeCore` and `AnankeGPU`, a CLI that runs and compares modes, and digest verification at every
stage.

**int-llm** is the C integer reference: a working scalar library with a determinism gate over a
4,352-point grid, an integer Llama inference path, and no GPU code at all.

**What each already answers, and this is the load-bearing part.** ANANKE demonstrates that the
CPU/GPU digest-equality *architecture* works on this hardware and that partition invariance is
achievable and testable — receipts, mode selection, worker and shard invariance, caps. ARCHE-Int
specifies the *arithmetic* that makes exact integer tensor work possible on Metal. int-llm has the
*reference semantics* and a real inference path.

**What none of them answers**, and it must not be assumed from the above: whether Metal can do
**two-limb 128-bit integer accumulation** at a cost worth paying. ANANKE's determinism is over
32-bit salience and evidence values through its own pipeline operations, **not** over Q16.48
fixed-point tensor arithmetic with 128-bit accumulators, so it is prior art for the harness and
the invariance discipline, **not evidence about the arithmetic**. ARCHE-INT-02's entire experiment
section exists to answer exactly that question and has not run.

**The inversion worth taking.** int-llm is C, builds today, and has a working gate. If it
implements ARCHE-Int's contracts, it becomes the **first execution of them** — and answers the
Metal feasibility question ARCHE-Int is itself blocked on. The dependency runs both ways, and that
is the argument for Ruling 1 rather than a cost of it.

---

## 3. Inheritance boundary

**From ARCHE-Int, taken as specification and not re-derived:** the Q16.48 format and its rounding
table; the reduction rule with its per-operation admission precondition read from each operand's
exact recorded extrema; the `W128` two-limb representation and its five routines with the
`W128Narrowed` return contract; the three GPU-partition conditions; the refusal-versus-saturation
posture; the ordered per-operator digest ladder and the DAG-shaped lesion rule.

**From ANANKE, taken as proven pattern and not as a dependency:** the receipt shape, digest-at-every-stage
discipline, explicit mode selection, and above all the **invariance test matrix** — one worker
against many, serial against parallel, CPU against GPU, all asserted equal by digest. That matrix
is the concrete form R3 must take here, and it exists and runs.

**int-llm owns:** the reference semantics every other implementation is measured against, the C
kernels, the fixture grid, and — new — the digest ladder over its own inference path.

**How drift is prevented, because two implementations of one specification is exactly how silent
divergence happens.** Every contract inherited from ARCHE-Int is cited by document and section at
the point of use, and any place int-llm must depart is recorded as a **declared divergence** with
its reason, in a register, the way ARCHE-Int already declares its departures from `int-llm`. An
undeclared difference between the two is a defect in whichever moved last.

---

## 4. Method: three classes of change, and what the golden can adjudicate

Ash's ruling 2 is right that behaviour changes move the hash. The discipline that survives it is
not "avoid moving it" but **one move, one reason** — because the hash's real job here is not to
block change, it is to *localise* it, and that is the instrument a determinism programme can least
afford to blunt.

**Class A — must not move the hash.** Relocating state, removing hidden globals. The hash is a
true oracle: green means correct, red means broken, no third reading. Batchable, because failures
bisect.

**Class B — will move the hash.** Widening an accumulator, precomputing constants, correcting a
narrowing. The hash is only a change *detector*. The proof is elsewhere — a directed test that
fails on the old code and passes on the new — and the golden is then re-blessed **deliberately, in
a commit that does nothing else**, so the new value has exactly one attributable cause.

**Class C — invisible to the hash.** `make determinism` covers `fp_math.h` only; `make regression`
adds `fp_test.c`'s unit tests and both determinism binaries; `make llama` is a manual command
needing a model directory. **No automated gate touches `llama_int.c`.** Every Class C change needs
a falsifier written *before* the fix, failing on current code.

**A fourth category arrives with this programme: cross-implementation equality.** CPU against GPU,
and int-llm against ARCHE-Int. Neither existing gate can express it, and §4's stages build it.

---

## 5. Stages

Each stage names the claim it establishes, its gate, and what it must not claim.

### S0 — Establish the baseline honestly

Run `make regression`, `make determinism`, `make determinism-portable` and record the passes and
the golden value **before touching anything**, so a later failure cannot be attributed to the
starting state. Record the toolchain, the compiler version and the machine. *Claims: R1 only.*

### S1 — Correct the CPU kernels (defects D4, D5, D6)

The three `llama_int.c` defects from the companion plan: the unchecked 128-to-64 narrowing at five
sites, the unchecked epsilon add in `rmsnorm`, and the 64-bit softmax accumulator whose guard is
reachable only by wrapping. **This is stage one and not stage three**, because a GPU implementation
cannot be proven bit-identical to a CPU reference that is itself silently wrong — you would be
certifying agreement with a defect.

Class C throughout: each fix needs its falsifier written first. D6 is Class B as well, since
widening the accumulator changes results above 32,767 terms, which is the point. *Claims: R1, and
correctness of the reference. Must not claim R3 or R4.*

### S2 — Remove hidden state (D1, D2, D3, and D7 below)

Explicit generator state with the `_r` pattern; the zero-seed guard; `fp_math_init`'s mutable
CORDIC and pi globals made constant or explicit. **R2 is unprovable while state is implicit and
per-translation-unit**, and R3 is unprovable while any of it is shared across workers.

**D7, new in this document: the build requires `-fwrapv`.** The Makefile compiles everything with
it (line 38) and states at line 32 that it is required, because the code relies on signed overflow
wrapping as two's complement. Signed overflow is undefined behaviour in standard C; `-fwrapv`
defines it. **A determinism claim that holds only under one compiler flag is not R4**, and the
softmax defect D6 is a live consequence of depending on wrapping rather than bounding the
arithmetic. The programme's target is to remove the *dependency* — bound or check every arithmetic
site so wrapping is never relied upon — and keep the flag afterwards as belt-and-braces rather than
as load-bearing. *Claims: R2. Opens the path to R4.*

### S3 — Build the digest ladder over the inference path

Port ARCHE-INT-04's mechanism: an ordered per-operator digest through each layer, with a model-level
and per-layer digest above it, each computed over the canonical byte stream **including shape and
scale metadata**, so a value that is right under the wrong shape still fails. Add the completeness
assertion — the number of recorded digests must equal a count computed from the config — and the
DAG-shaped lesion rule: neutering one operator must change its own index and every causal
descendant, and **leave every non-descendant bit-identical**.

**This stage is the instrument everything after it depends on.** Without it, a CPU/GPU divergence
tells you the model output differs and nothing about where. With it, the first mismatching index
names the operator. It also closes the Class C coverage gap by giving `llama_int.c` its first
automated gate. *Claims: nothing new by itself; it is the measuring instrument.*

### S4 — Freeze the CPU oracle and adopt the two-verdict structure

With S1 corrected and S3 measuring, the CPU path becomes the reference every other arm is compared
against. Adopt ARCHE-Int's separation explicitly: a **reference-fidelity** verdict, promoted by the
external fixture grid, and a **mathematical-validity** verdict, promoted by independent
exact-rational fixtures — never by the grid, because a faithfully reproduced reference defect
passes it. Where the two conflict, mathematical validity governs what ships and the divergence is
declared with its reason. *Claims: R1, R2 across the whole inference path.*

### S5 — Metal integer kernels

Implement ARCHE-INT-02's specified contracts: the two-limb `W128` accumulator with its five
routines, the checked narrow returning both value and range flag, and the exact-arm gate of **zero
differing elements** — any divergence is a failure, not a tolerance. Follow ANANKE's harness shape
for mode selection and receipts rather than inventing one.

**The three partition conditions are the whole of R3 and must be honoured literally:** every
partial accumulator is full-width and never rescaled or narrowed; the single rescale and narrow
happen exactly once on the fully combined sum; and no partial can overflow, which the admission
precondition already guarantees since every partial is bounded by the same sum of absolute products
that admitted the row. A partial that rescales reintroduces order dependence, and that is the one
mistake that makes hybrid execution unprovable.

**The honest unknown:** whether MSL's 64-bit integer arithmetic makes this fast enough to be worth
doing is unmeasured, here and in ARCHE-Int. **A negative result is a valid outcome of this stage**,
not a failure of the programme — the CPU oracle survives as the deliverable, and the finding is
worth having. *Claims: R3 across threadgroup and partition shape, GPU-internal.*

### S6 — Hybrid CPU and GPU execution

Split one reduction across both, combining full-width partials. Provable **only** under S5's three
conditions, and cheap once they hold, because exact integer addition is associative and commutative
so any partition of the same pairs yields identical bits. The split ratio becomes a receipt field,
explicitly **not** a digest input — the same treatment ARCHE-Int gives tiling boundaries and
threadgroup shape.

The falsifier is the one that matters: identical digests across at least five split ratios
including all-CPU and all-GPU, plus a deliberately corrupted arm that must diverge, so the harness
is shown capable of detecting a difference at all. *Claims: R3 in full — the programme's headline.*

### S7 — Platform invariance

Only after S2 removes the wrapping dependency. Extend `determinism-portable` into a matrix across
architectures, and state plainly what is and is not claimed: bit-identity across machines of the
same architecture is achievable with pure bounded integer arithmetic; across architectures it holds
only where every operation is specified to the bit, which is what the seventeen-rule ABI exists to
do. *Claims: R4, scoped to what was actually tested.*

---

## 6. Open decisions, all owed before the stage that needs them

**D2's zero seed:** reject or normalise. Recommendation normalise, because rejecting puts an error
path into every caller for a case only corrupt input produces. *Owed before S2.*

**D4's out-of-range narrow:** refuse, saturate, or record-and-continue — and it must be the same at
all five sites. This is the single most consequential choice in the programme, because it decides
what the engine does when the arithmetic runs out of room, and every later arm inherits it.
*Owed before S1.*

**Whether the exact arm is the only arm.** ARCHE-Int keeps a practical, lower-precision Metal arm
as a separate candidate beside the exact one. Whether this programme wants that, or exact-only,
decides how much of S5 exists. *Owed before S5.*

**How far ANANKE is reused.** Pattern only, as assumed here, or an actual dependency for the
receipt and invariance machinery. A dependency buys working code and costs a Swift boundary in a C
project. *Owed before S3.*

---

## 7. Risks

**Three repositories converging on one problem is the programme's largest risk, not its largest
asset.** ARCHE-Int specifies but has never run; ANANKE runs but over a different computation;
int-llm runs but has no GPU. The failure mode is three partial answers that never compose, each
believing the others cover what it does not. The mitigation is the declared-divergence register in
§3 and nothing softer.

**S1 before S5 is not negotiable.** Certifying that a GPU agrees with a CPU reference that
silently truncates would produce a bit-identical wrong answer on two devices and call it
determinism.

**The golden will move several times.** Each move must have exactly one attributable cause and its
own commit. A batch of behaviour changes followed by one re-bless produces a reference value nobody
can account for, and every subsequent run inherits it.

**Nothing in this plan has been executed.** Line numbers, flags, constants and repository states
were read; `make` was not invoked, and no ARCHE-Int or ANANKE test was run. The first act of stage
zero is to make that false.

---

## 8. Provenance

`int-llm` facts read at `0b4b6d04eb3e9e969d125804a154309eed3be9de`: the Makefile's flags and
targets, `fp_math.h`'s state and initialisation, `llama_int.c`'s three kernel defects, and the
golden value. ARCHE-Int contracts are cited from the blueprint family at the head of
`plumbline/arche-int-plans` and are **specification, not executed behaviour**. **ANANKE was tested on 2026-08-18 rather than
taken on trust, and the result narrows what §2 and §3 may claim from it.**

*What ran, on an Apple M3 Max, macOS 26.5.2, arm64, `ananke 0.1.0`, Metal reported available:*
`swift build` clean; `swift test` **76 tests, 0 failures**; and the README's `run` invocation
across six configurations — CPU and GPU modes, one worker and eight and three, parallel on and
off — all returning the **identical** digest `83ec0ad8…f97a1bb2`. A negative control with the seed
changed from 42 to 43 returns a different digest, so the harness can detect a difference.

*What that does and does not establish, and the distinction is load-bearing.* The CLI's own usage
line reads "Run deterministic CPU or **GPU-generated** pipeline", and the code path matches:
`PipelineMode.gpu` requires a **work-item generator** and `saturate` calls
`SalienceDeltaGeneratorGPU.generateWorkItems` followed by `CPUPipelineRunner.runPipelineOnce`. So
what is demonstrated is that **Metal-generated work items produce a digest identical to
CPU-generated ones**, with the pipeline computation itself running on CPU in both modes. That is a
genuine and useful result — Metal compute producing bit-identical output to a CPU path on this
hardware — and it is **narrower than "the same pipeline runs on CPU or GPU"**, which is how the
`README.md` states it and how an earlier revision of this plan repeated it.

*Bound on this finding:* read from the CLI help, two call sites and a fragment of
`SaliencePipeline.swift` under a thinned context, and confirmed by run rather than by audit. **The
Metal-side share of the work should be re-derived before S5 depends on it.** The working copy at
`$HOME/GitHub/ANANKE` is **not a git repository**, so this result is pinned to no commit — pin it
before citing it.

Nothing else here was executed: no `make` in this repository, and no ARCHE-Int test.

---

*Straightedge (Claude Opus 5), contributor 35, 2026-08-18. Programme plan only. Two decisions are
owed before stage one begins, and the ANANKE claim in §2 is unverified prior art rather than
evidence.*
