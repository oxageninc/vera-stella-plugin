# Vera

The verification engine for [Stella](https://github.com/oxageninc/stella), as a
plugin rather than a core subsystem.

Vera's job is narrow and it is the whole job: **author a test that fails on the
old code and passes on the new one, decide the flip deterministically, and stamp
that decision into the trace in a form a fine-tuning pipeline can consume.**

Everything Stella's staged pipeline does that is not that — triage, recall,
research, planning, scoping, candidate fan-out, model rostering — stays behind.
Vera is a port of the verification nucleus, not a lift of the pipeline crate.

## Why this repo exists

`stella-core` is meant to be a bare turn loop: minimal tools, one model,
transparent, no I/O. The witness protocol, the verification ladder and the flip
oracle were a large opinionated policy layer grown into that loop. Extracting
them forces the plugin surface to become genuinely capable — more than one
hard-coded pipeline mode and one hard-coded witness protocol — which is what
lets a customer define *done* for themselves.

## What Vera is

A **wrapper plugin** against Stella's four-point wrapper contract:

| Point | Vera's use |
|---|---|
| `before_turn` | nothing (Vera does not plan work) |
| `after_turn` | author the witness, run it, gather evidence |
| `judge` | the verification ladder and the flip oracle — **calls no model** |
| `again?` | demand evidence or revise, or stop with an outcome |

`judge` calling no model is a structural rule inherited from Stella, not a
default: the ladder is terminal at every outcome and the flip is decided by
running the test, never by asking one.

The single model call Vera makes is the **witness author**, in `after_turn`. It
survives because it builds the measuring stick rather than substituting for one.

## The port surface

The nucleus is already pure functions over owned data, which is what makes a
port realistic rather than a rewrite. From `stella-pipeline`:

**Witness authoring**
- `witness.rs` — prompt construction, `parse_test_invocation`, `runner_probe`,
  and the three acceptance validators (artifact, invocation, identity).
- `witness/airlock.rs` — `DisclosureGrain`, `SymptomClass`, `FailureFingerprint`,
  scrub/redact.
- `pipeline/witness_stage.rs` — author, one bounded repair, acceptance.

**Flip oracle and ladder**
- `verify.rs` — `FlipOracle`, `FlipState`, `ObserveOutcome`, `ladder_decision`
  with `LadderInputs`/`LadderDecision`, the evidence builders, and
  `strip_witness_hunks` (tamper exclusion).
- `flip_halt.rs` — the fail→pass transition observer.

**The invariant to carry over first:** the property test
`flip_requires_a_prior_failing_observation`. A flip with no prior failing
observation is not a flip.

**Known entanglement:** tamper exclusion currently lives inside
`Pipeline::verify_candidate` in the pipeline's god file rather than beside the
oracle. It is the one piece that must be lifted out rather than copied across.

## The half that is net-new

Trace stamping is **not** a port — it does not exist upstream yet.

Today a `LadderSnapshot` (tri-state `flip: FlipOutcome`, `unstable_flip`,
`flip_refused_different_failure`, `verify_done_flip`) rides inside
`AgentEvent::Verdict`. But there is no dedicated flip-transition event, and by
Stella's own consumer ledger the `verdict` tag's posture is `Unclassified` —
nothing is declared to read it. The only durable consumers are rendering and
export.

So Vera owns a **durable, declared flip record** as a first-class emission with
a named consumer: the fine-tuning corpus. A verification signal that nothing
reads is the exact failure mode this project exists to end.

## Upstream status this depends on

- **Landed** — the engine owns its own ending; the pipeline no longer holds a
  private event channel into the loop. This was the hard blocker: a plugin
  cannot hold such a channel.
- **Landed but inert** — the `[wrapper]` TOML manifest (stage names, signals,
  a closed condition grammar) parses and load-checks, but nothing binds a stage
  to the loop yet.
- **Not started** — the four-point wrapper contract itself. This is the socket
  Vera plugs into, and Vera cannot run in-loop until it exists.

Consequence: Vera can be designed, ported and unit-tested against its own pure
core *now*. It cannot be wired end-to-end until the wrapper socket lands.

## Status

Charter only. No code yet.
