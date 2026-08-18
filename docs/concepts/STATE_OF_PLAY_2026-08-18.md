# State of play at the ANANKE pivot — 2026-08-18

Written by Straightedge (Claude Opus 5), contributor 35, at Ash's instruction to document
exactly where everything stands before changing direction to ANANKE. **Three lanes are open.
Two are paused here in a clean state; the third is where work moves next.**

---

## 1. ARCHE-Int — paused, complete for this seat, all pushed

Branch `plumbline/arche-int-plans` in the ARCHE repo. Every commit pushed;
`python3 scripts/arche_int_freeze.py verify` prints `HANDOFF_INTEGRITY=PASS` at head with 27
artefacts pinned, and the freeze cycle was run after **every** repair commit, so any commit on
that branch is a valid send point.

**Done.** Datum's repair steps 2, 3 and 4 — ARCHE-INT-02, -03 and -04 repaired against Eko's
round-6 disposition record, with ARCHE-INT-01 corrected where its text was half of a defect.
Fifteen of Eko's twenty findings are now repaired. Three read-only adversarial reviewers were
raised against those repairs and found **eleven defects in them**, all applied — including a
falsifier no conforming implementation could have passed. A build plan for the eight unspecified
scalar operators exists with **all three Phase 0 questions ruled by Ash**.

**Open, and none of it is blocked on me.** BP-16 and BP-17 wait on ARCHE-INT-05; BP-02's
ARCHE-INT-00 half; steps 5 through 8 (`05`, `06`, the branch split, `07`'s remainder); and step 9,
a fresh independent re-judgement. **Nothing was sent to Eko** — Ash's sequencing condition is now
satisfied, so that send is his to make.

**BP-20 is the largest thing outstanding and no repair touched it:** no ARCHE build or test has
been run at any point in that family's authoring, across six rounds.

**Two items for Ash.** ARCHE-INT-01's acceptance criterion and VM-01 are unsatisfiable until the
eight operators land — the interim correction is in the build plan §6 and should go in first. And
all eight blueprint headers say Slipgauge authored as Claude Opus 5 while their own handoff and
eleven of fourteen commits say Claude Fable 5, with three the same day saying Opus 5; one signing
is wrong and the repo cannot say which. That bears on a KANON cross-substrate gate.

**Owed and small:** ARCHE-INT-04's filename and title still say "Isomorphic" while its §1
withdraws the claim. Rename at the next reissue together with ARCHE-INT-00's family table and the
freeze script's declared membership.

---

## 2. int-llm — paused, plans only, no code written

Branch `straightedge/rng-state-carried-plan` in `kryptosmatrix/int-llm`, pushed. **No source file
was modified and nothing was built or run in this repository.**

Two documents. `RNG_STATE_CARRIED_UPGRADE_PLAN.md` is a seven-defect register: the generator's
file-static global; xorshift64's absorbing zero state with no seed validation, restorable from a
corrupt checkpoint; `fp_math_init`'s runtime-computed mutable CORDIC and pi globals; the unchecked
128-to-64 narrowing at five sites in the matrix multiply; `rmsnorm`'s unchecked epsilon add; the
64-bit softmax accumulator whose guard is reachable only by wrapping; and **D7**, the build's
required `-fwrapv`, which makes the determinism conditional on one compiler flag.

`DETERMINISTIC_CPU_GPU_PROGRAMME.md` is the eight-stage programme for Ash's larger goal, under his
ruling that it **inherits ARCHE-Int's specification** rather than deriving its own.

**Four decisions owed before work starts**, each before its stage: the zero-seed behaviour
(recommend normalise); the out-of-range narrow behaviour, **the most consequential choice in the
programme**, since it decides what the engine does when arithmetic runs out of room and every
later arm inherits it; whether an exact arm is the only arm; and how far ANANKE is reused.

**One finding worth carrying into the ANANKE work:** the determinism gate covers `fp_math.h` only.
No automated target touches `llama_int.c`, where three of the seven defects live.

---

## 3. ANANKE — where work moves next

Tested by run on 2026-08-18, Apple M3 Max, macOS 26.5.2, `ananke 0.1.0`, Metal reported
available. `swift build` clean. `swift test` 76 tests, 0 failures. The README's own `run`
invocation across six configurations — CPU and GPU modes, one worker, three and eight, parallel on
and off — returned the **identical** digest `83ec0ad8…f97a1bb2`. A seed change from 42 to 43 moves
the digest, so the harness can detect a difference.

**The README's central claim is inaccurate as written, and that is a defect rather than a scope
note.** It states "run the same pipeline with the same inputs on CPU or GPU" and "GPU digest
matches CPU digest". The CLI's own usage line says "CPU or **GPU-generated** pipeline", and the
code agrees: `PipelineMode.gpu` requires a work-item **generator**, and the saturation path calls
`SalienceDeltaGeneratorGPU.generateWorkItems` and then `CPUPipelineRunner.runPipelineOnce`. What is
demonstrated is that **Metal-generated work items produce a digest identical to CPU-generated
ones, with the pipeline computation running on CPU in both modes.** A reader would reasonably
conclude the pipeline executes on GPU. It does not, on this evidence.

**Confidence: moderate, not high.** Read from the CLI help, two call sites and a fragment of
`SaliencePipeline.swift` under a thinned context, confirmed **by run rather than by audit**. There
could be GPU stages not seen. **No audit of ANANKE has been performed and no defect in its code
has been found** — the finding above is about the README against the code's shape.

**Housekeeping:** the working copy at `$HOME/GitHub/ANANKE` is **not a git repository**, despite
the README naming a GitHub package URL. Every result above is therefore pinned to no commit. Pin
it before citing it.

---

## 4. The gap that motivates the pivot

**Nothing in the estate has demonstrated GPU arithmetic bit-identical to a CPU oracle.** ANANKE
demonstrates a Metal *generation* stage agreeing with a CPU one. ARCHE-INT-02 specifies the
experiment that would answer the real question — whether Metal can do two-limb 128-bit integer
accumulation at a cost worth paying — and has never run. int-llm has no GPU code at all.

That is the hole all three lanes have been circling, and it is why ANANKE is the right place to
work next: it is the only one of the three that **builds and runs today** with a Metal path
already wired, so it is the cheapest place to find out what Apple Silicon will actually do.

---

## 5. Where to resume each lane

ARCHE-Int: read `ARCHE_INT_HANDOFF_2026-08-18_DATUM.md`, then Slipgauge's transport handoff, then
`ARCHE_INT_HANDOFF_2026-08-18_STRAIGHTEDGE.md`. Repair from the round-6 disposition record, whose
§9 records where the applied repairs departed from what it adopted. Run the freeze cycle's
`verify` before believing any hand-off.

int-llm: this branch, the two plans, and the four owed decisions. Stage zero is to run the gate and
record the pass before touching anything.

ANANKE: start from §3 above and treat every line of it as carried-not-verified. The first honest
act is an audit of how much of the pipeline is GPU-side, because §3's central finding is the thing
most likely to be wrong.

---

*Straightedge (Claude Opus 5), contributor 35, 2026-08-18, Australia/Brisbane.*
