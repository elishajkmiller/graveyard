# Kelvin
## Language Specification — v0.7

---

## What it is

A quantum programming language where physical hardware constraints are
first-class typed constructs, not runtime surprises.

QPUs fail in physically predictable ways. Qubits decohere when temperature
rises. Gate fidelity degrades as thermal noise increases. Coherence time
bounds how long a computation can run before measurement becomes
unreliable. Every existing quantum language treats these constraints as
external concerns — things you handle in infrastructure, not in code.
Kelvin treats them as part of the type system.

---

## How Kelvin differs from existing languages

**Q#** has linear types for qubits. It does not have physical layer
awareness. Physical constraints in Q# live outside the language — in
calibration scripts and hardware dashboards. Kelvin makes them part
of the program.

**Silq** has automatic uncomputation — the compiler generates reverse
operations to clean up ancilla qubits. Silq does not have physical layer
awareness. It targets algorithm expression. Kelvin targets hardware
execution. These are complementary, not competing.

**Quipper** is the most expressive language for circuit construction.
It targets circuit description. The gap between a Quipper circuit and
running on real hardware requires infrastructure Quipper does not address.
Kelvin targets that execution layer.

**OpenQASM 3.0** is assembly. Writing it directly is like writing x86
for a web server. It is the correct compilation target, not the correct
programming interface. Kelvin compiles to OpenQASM 3.0.

**The gap Kelvin fills:** no existing language lets you express fidelity
requirements, thermal adaptation, and qubit ownership as a unified,
compiler-checked program. That is what Kelvin does.

---

## The physical foundation

QPUs operate at approximately 15 millikelvin — colder than outer space.
The name Kelvin reflects this directly.

Thermal stability is a correctness concern, not a deployment concern.
A qubit's coherence time — how long it maintains superposition before
environmental noise collapses it — is a function of temperature. A
computation that exceeds its qubits' coherence time produces wrong
answers that look like right answers. The hardware does not raise an
error. Kelvin surfaces this in the type system.

Three physical facts the language is built on:

**Qubits cannot be copied.** The no-cloning theorem is a law of physics.
The type system enforces it — clone is not defined on Qubit.

**Measurement is destructive.** Measuring a qubit collapses its wave
function. The state is gone. Kelvin's linear type system enforces this —
reading a measured qubit is a compile-time error.

**Measurement disturbs.** The act of measuring changes what is measured.
This is established. Why it happens is genuinely open physics. Kelvin
exposes the disturbance without claiming to explain it.

---

## Types

### Numeric precision

`Amplitude` : complex float64 (real, imaginary)
`Frequency`  : float64, positive
`Ratio`      : float64, clamped [0.0, 1.0]
`Collapsed`  : uint64, measured bit string as integer
`Count`      : uint64, non-negative integer
`Interval`   : (lower: float64, upper: float64, delta: float64)

### Quantum types

| Type | What it is |
|---|---|
| `Qubit` | single qubit in superposition, linear ownership |
| `Entangled(n)` | n correlated qubits, linear ownership, single unit |
| `Collapsed` | classical value after measurement |
| `Amplitude` | complex probability weight |
| `Distribution` | measurement result: counts, probs, entropy, most_probable |

**Qubit lifecycle — compiler enforced:**

    created → superposed → [entangled] → measured → consumed

After measure, a qubit is consumed. ReadAfterConsume is a compile-time
error where statically detectable, runtime error otherwise.

### Classical types

| Type | What it is |
|---|---|
| `Frequency` | positive real — rate, physical measurement |
| `Count` | non-negative integer |
| `Ratio` | real in [0.0, 1.0] |
| `Text` | UTF-8 byte sequence |
| `Flag` | true or false |
| `Silence` | absence of a value |
| `Interval` | lower, upper, delta — result of all measure calls |

### Struct types

    struct Result:
      answer:     Collapsed
      confidence: Ratio
      shots:      Count

    let r ← Result(answer: 3, confidence: 0.94, shots: 1000)
    let r ← r with confidence: 0.97    ~ non-destructive update

---

## Part 1 — Ownership Model

### Rules

1. Every qubit has exactly one owner at any point in time.
2. Ownership can be moved. It cannot be copied.
3. A qubit can be borrowed for operations that do not measure it.
   Borrowing does not transfer ownership.

### Move

    fn prepare(): Qubit
      qubit q
      interfere q: hadamard
      return q              ~ ownership moves to caller

    fn consume(q: Qubit): Collapsed
      return measure q      ~ q consumed, ownership ends

    let q ← run prepare()
    let r ← run consume(q)
    let x ← q              ~ ERROR: UseAfterMove

### Borrow

A function taking `&Qubit` borrows without consuming. The compiler
verifies borrowed qubits are not measured inside the borrowing function.

    fn inspect(q: &Qubit): Amplitude
      return amplitude(q)    ~ read amplitude — allowed

    fn illegal(q: &Qubit): Collapsed
      return measure q       ~ ERROR: cannot measure borrowed qubit

    let q ← qubit
    let a ← run inspect(&q)  ~ q still owned
    let r ← measure q         ~ q consumed

### Entanglement ownership

`entangle` consumes its inputs and produces a new `Entangled(n)`.
Original qubits cease to exist as independent values.

    qubit a, b
    let pair ← entangle a, b
    ~ a and b consumed, pair : Entangled(2)
    let x ← a              ~ ERROR: UseAfterMove

Entangled values cannot be partially moved. The entire `Entangled(n)`
is owned, borrowed, or consumed as a unit.

### Inference rules

    Γ ⊢ q : Qubit
    ─────────────────────────── [Move]
    Γ, q consumed ⊢ f(q) : T

    Γ ⊢ q : Qubit    f : &Qubit → T    f does not measure
    ──────────────────────────────────────────────────────── [Borrow]
    Γ ⊢ f(&q) : T    q remains in Γ

    Γ ⊢ a : Qubit    Γ ⊢ b : Qubit
    ───────────────────────────────────────────────── [Entangle]
    Γ, a consumed, b consumed ⊢ entangle(a,b) : Entangled(2)

    Γ ⊢ q : Qubit
    ─────────────────────────────────── [Measure]
    Γ, q consumed ⊢ measure q : Collapsed

    Γ ⊢ e : Entangled(n)
    ──────────────────────────────────────────────── [MeasureAll]
    Γ, e consumed ⊢ measure e : [Collapsed; n]

    Γ ⊢ e : Entangled(n)    0 ≤ i < n
    ──────────────────────────────────────────────────────────── [Split]
    Γ, e consumed ⊢ split e measure: i : (Collapsed, Qubit)

    Γ ⊢ e : Entangled(n)    0 ≤ i < n
    ──────────────────────────────────────────────────────────── [Marginalize]
    Γ, e consumed ⊢ measure e marginalize: i : Distribution

    Γ ⊢ b₁ ... bₙ — parallel branches with disjoint qubit ownership
    ──────────────────────────────────────────────────────────────── [Parallel]
    Γ ⊢ parallel: b₁ ... bₙ join : (T₁, ..., Tₙ)

    Γ ⊢ sensor — physical sensor expression
    Γ ⊢ body assigns priority : Ratio
    body does not measure any Qubit or Entangled(n)
    ─────────────────────────────────────────────── [Adapt]
    Γ ⊢ adapt sensor → priority backoff: policy: body : Silence

### Clone is not defined

    let q2 ← clone(q)    ~ ERROR: Qubit does not implement Clone

### RAII and scope exit

Qubits not consumed at scope exit are automatically released.
Release requires a measurement — this is a physical fact, not a
language policy. The compiler warns:

    fn leaky(): Silence
      qubit q
      interfere q: hadamard
      ~ WARNING: implicit release of q at scope exit — measurement discarded

To suppress:

    drop q    ~ explicit measure and discard, no warning

---

## Part 2 — Measure Semantics

### Single qubit

    let r ← measure q
    ~ q consumed, r : Collapsed

    let r ← measure q shots: N
    ~ q consumed, r : Distribution
    ~ r.samples        : [Collapsed; N]
    ~ r.counts         : Map(Collapsed, Count)
    ~ r.probs          : Map(Collapsed, Ratio)
    ~ r.most_probable  : Collapsed
    ~ r.entropy        : float64

### Joint measurement

Measuring `Entangled(n)` returns the joint distribution over all n qubits
as a single bit string. Bit ordering is most-significant-first.

    qubit a, b
    let pair ← entangle a, b
    let r ← measure pair shots: 1000
    ~ r : Distribution over {00, 01, 10, 11}
    ~ Bell state: ~500x "00", ~500x "11", ~0x "01", ~0x "10"

### Marginal measurement

Returns the marginal distribution for one qubit index. Entangled state
is fully consumed. Does not return a live qubit.

    let r ← measure pair marginalize: 0 shots: 1000
    ~ r : Distribution over {0, 1}
    ~ pair consumed entirely

### Partial measurement — split

Measures one qubit from an entangled pair and returns the remaining
qubit as a live, collapsed (definite-state) Qubit.

    let (flag, remaining) ← split pair measure: 0
    ~ flag      : Collapsed
    ~ remaining : Qubit — definite state, correlated with flag
    ~ pair consumed

    ~ remaining is live — can apply gates and measure
    interfere remaining: phase(flag)
    let final ← measure remaining

### Concurrent QPU access

Parallel branches own disjoint sets of qubits. No aliasing of quantum
state across branches is permitted — the compiler verifies disjoint
ownership at the parallel boundary.

    parallel:
      let r1 ← run routine_a(data_1)    ~ owns its qubits
      let r2 ← run routine_b(data_2)    ~ owns its qubits
    join
    ~ r1, r2 : classical values

---

## Part 3 — Resource Management

### Allocation

    qubit a              ~ one physical qubit allocated from QPU pool
    qubit a, b, c        ~ three allocated
    qubit a ← superpose(data)   ~ initialized from classical data

If the QPU pool cannot satisfy the request:

    error CapabilityMissing(e):
      ~ e.requested : Count
      ~ e.available : Count

### Release

Qubits are released when consumed: via `measure`, via `drop`, or
implicitly at scope exit with compiler warning.

Release returns the physical qubit to the QPU pool immediately.
Physical qubits are scarce — a typical QPU has 100–1000 total.
Deterministic release is essential.

### Reuse after measure

A consumed qubit variable cannot be reused. The physical qubit
returns to the pool and may be reallocated, but the variable is gone.

    let r ← measure q         ~ q consumed, physical qubit returns to pool
    qubit q2                   ~ fresh allocation — language model: unrelated
                               ~ may happen to get same physical qubit

### Error propagation across parallel branches

If one branch in a parallel block raises an error before join:

1. All qubits owned by the erroring branch are dropped in declaration
   order — each drop causes an implicit measure and discard.
2. All other branches are signaled to halt at their next safe point.
3. Each halting branch drops its qubits in declaration order.
4. The error surfaces at the join point to the enclosing error handler.
5. No partial results from any branch are visible after the error.

This rule is formal — it is not implementation-defined behavior.

    parallel:
      let r1 ← run routine_a()    ~ if this errors:
      let r2 ← run routine_b()    ~ this branch halts, qubits dropped
    join

    error RoutineError(e):
      ~ both branches fully cleaned up before reaching here
      ~ no qubit leaks, no partial state

---

## Part 4 — Physical Layer

This is the central contribution. Not ownership — ownership is a solved
problem. The physical layer is what no other quantum language has.

### capability — query before use

    let has_qpu   ← capability quantum             ~ Flag
    let qubits    ← capability quantum.qubits      ~ Count
    let fidelity  ← capability quantum.fidelity    ~ Ratio
    let coherence ← capability quantum.coherence   ~ Frequency (microseconds)
    let has_therm ← capability thermal             ~ Flag
    let settling  ← capability thermal.settling    ~ Frequency (settling time)
    let timing_δ  ← capability timing.delta        ~ Frequency (precision)

Declare requirements — error at startup if unmet:

    require quantum qubits: 8
    require quantum fidelity: 0.999
    prefer  thermal

### measure — read physical state

Returns Interval (lower, upper, delta) for all physical measurements.

    let temp  ← measure thermal    ~ millikelvin for QPU hardware
    let load  ← measure cpu        ~ Ratio
    let drift ← measure context    ~ Frequency, accumulated drift

### hint — physical guidance to runtime

    hint thermal: stable    ~ request stable temperature for coherence
    hint all:     open      ~ no restrictions

Hints. Not commands. The runtime makes best effort.

### adapt — typed feedback with formal contract

    adapt <sensor> → priority backoff: <policy>(<params>): <body>

**Types:**
- `sensor`   : `thermal` | `cpu` | `radio` | `context`
- `priority` : Ratio assigned in body
- `policy`   : `exponential` | `linear`
- `exponential` params: `min: float64, max: float64` (seconds)
- `linear` params: `min: float64, max: float64, step: float64`

**Execution model:**
1. Measure sensor. Record in execution log.
2. Execute body. Record new priority.
3. Apply priority to runtime scheduler.
4. Compute next poll interval per backoff policy.
5. If priority unchanged: backoff increases. If changed: reset to min.

**Oscillation enforcement:**
The compiler detects the simple threshold pattern:

    let priority ← sensor_value > K ? A : B

where A ≠ B. If `backoff.min` < `capability thermal.settling`, the
compiler emits a warning. The programmer may suppress with explicit
acknowledgment:

    adapt thermal → priority
      backoff: exponential(min: 5.0, max: 60.0)
      suppress: oscillation:
      ...

Runtime enforcement: the performer tracks priority history over a
sliding window. If priority alternates between two values more than
3 times within one `backoff.max` interval, the performer forces
backoff to `max` until the pattern stabilizes. This is logged.

Deeper oscillation analysis — CFG-based, abstract interpretation —
is future work. The spec does not promise it.

---

## Part 5 — interfere Instruction Set

| Kelvin | OpenQASM 3.0 | Description |
|---|---|---|
| `interfere q: hadamard` | `h q` | Equal superposition |
| `interfere q: phase(θ)` | `p(θ) q` | Phase rotation, θ in float64 radians |
| `interfere q: not` | `x q` | Bit flip |
| `interfere q: phase_flip` | `z q` | Phase flip |
| `interfere q, c: controlled` | `cx c, q` | CNOT |
| `interfere q: rotate_x(θ)` | `rx(θ) q` | X-axis rotation |
| `interfere q: rotate_y(θ)` | `ry(θ) q` | Y-axis rotation |
| `interfere q: rotate_z(θ)` | `rz(θ) q` | Z-axis rotation |
| `interfere q: custom(U)` | `unitary(U) q` | Custom 2×2 unitary |

Custom unitary matrix syntax:

    interfere q: custom([[c64(0.707, 0.0), c64(0.707, 0.0)],
                         [c64(0.707, 0.0), c64(-0.707, 0.0)]])

`c64(real, imag)` — complex float64.

High-level shortcuts `toward` and `away` compile to amplitude
amplification sequences. The generated gate sequence is inspectable:

    musi analyze hello --gates

---

## Part 6 — Classical Infrastructure

### let — bind a value

    let x ← compute()
    let x: Frequency ← 440.0

Bindings are immutable. Rebinding shadows.

### fn — define a function

    fn name(params): ReturnType
      ~ body
      return value

### run — call a function

    let r ← run name(args)

### parallel — concurrent work

    parallel:
      let a ← run routine_a()
      let b ← run routine_b()
    join

Branches own disjoint state. No shared mutable bindings.
Results visible after join. Error propagation defined in Part 3.

### struct — data structure

    struct Name:
      field: Type

    let s ← Name(field: value)
    let s ← s with field: value    ~ non-destructive update

### loop — iteration

    loop 10:
      let i ← i + 1

    loop while condition:
      let x ← step()

### error — exception handling

    error Type(e):
      ~ handler body
      return value

    error any(e):
      log(e)
      return Silence

### drop — explicit release

    drop q    ~ qubit: measure and discard
    drop x    ~ classical: release memory
              ~ ReadAfterDrop is a runtime error

### scope — explicit scope block

    scope:
      let local ← 42
    ~ local consumed here

### rest — deliberate pause

    rest ≈500ms           ~ interval: [500ms − δ, 500ms + δ]
    rest ≈500ms ± 10ms    ~ explicit tolerance, must be ≥ δ(h)

---

## Part 7 — Execution Log Schema v1

Newline-delimited JSON. Each line is one event. Schema is versioned.
Replay requires exact schema match between producer and consumer.

### Header

```json
{
  "schema": "kelvin-log/1.0",
  "compiler_version": "0.7.0",
  "runtime_version": "0.7.0",
  "hardware_id": "ibm_nairobi_anon_a3f9",
  "context": "quantum",
  "rng_seed": 4792183641,
  "timestamp_resolution_ns": 1,
  "qubit_map": {
    "logical_0": "physical_3",
    "logical_1": "physical_7"
  },
  "capabilities": {
    "quantum_qubits": 7,
    "quantum_fidelity": 0.9987,
    "quantum_coherence_us": 120.4,
    "thermal": true,
    "thermal_settling_s": 3.2,
    "timing_delta_ns": 500
  },
  "started_at": "2025-05-03T14:22:01.000000000Z"
}
```

### Qubit allocation

```json
{
  "event": "qubit_alloc",
  "t_ns": 1024,
  "logical": ["q0", "q1"],
  "physical": ["3", "7"]
}
```

### Gate

```json
{
  "event": "gate",
  "t_ns": 2048,
  "gate": "h",
  "logical": "q0",
  "physical": "3",
  "openqasm": "h q[3];"
}
```

### Measure

```json
{
  "event": "measure",
  "t_ns": 8192,
  "logical": ["q0", "q1"],
  "physical": ["3", "7"],
  "shots": 1000,
  "joint": true,
  "marginalize": null,
  "split_index": null,
  "results": {
    "counts": {"0": 498, "3": 502},
    "probs":  {"0": 0.498, "3": 0.502},
    "most_probable": 3,
    "entropy": 0.9999
  },
  "bit_ordering": "msb_first"
}
```

Counts keyed by integer value of bit string. "0" = "00", "3" = "11"
for 2-qubit joint measurement.

### Thermal

```json
{
  "event": "adapt",
  "t_ns": 16384,
  "sensor": "thermal",
  "interval_lower": 14.998,
  "interval_upper": 15.002,
  "interval_delta": 0.004,
  "unit": "millikelvin",
  "priority_before": 0.8,
  "priority_after": 0.8,
  "backoff_next_ms": 1000,
  "oscillation_count": 0
}
```

### Error

```json
{
  "event": "error",
  "t_ns": 32768,
  "type": "CapabilityMissing",
  "message": "quantum qubits requested: 8, available: 7",
  "handled": true,
  "handler": "classical_fallback",
  "qubits_dropped": []
}
```

### Qubit release

```json
{
  "event": "qubit_release",
  "t_ns": 40960,
  "logical": ["q0", "q1"],
  "physical": ["3", "7"],
  "reason": "measure"
}
```

### Footer

```json
{
  "event": "finished",
  "t_ns": 65536,
  "exit": "clean",
  "total_qubits_allocated": 2,
  "total_shots": 1000,
  "total_gates": 2,
  "total_adapt_events": 3,
  "oscillation_enforcements": 0
}
```

### Replay determinism requirements

Replay uses recorded values — it does not re-run on hardware.
These fields must match exactly between original and replay execution:
`schema`, `rng_seed`, `qubit_map`, all `measure.results`,
all `adapt.interval_*` values.

`musi replay` refuses with a schema error on any mismatch.

---

## Part 8 — Compiler Model

    source.kv → compiler → score → runtime → execution

    musi build hello.kv -o hello
    musi run hello
    musi run hello --shadow         ~ read-only observer, shadow log
    musi eval hello.kv              ~ interpreted, no compilation
    musi eval hello.kv --simulate   ~ synthetic hardware, no QPU needed
    musi analyze hello --gates      ~ show generated OpenQASM gate sequences
    musi analyze hello --emit thermal
    musi replay execution.log

Source files use `.kv` extension.
Quantum backend: OpenQASM 3.0.
Hardware: IBM Quantum, Google Cirq, AWS Braket — no vendor lock-in.

### Debugging modes

**simulate** — all hardware synthetic. Quantum operations use classical
simulation. No real hardware touched. For development and CI.

**shadow** — read-only observer runs alongside real runtime. Cannot write,
cannot error, cannot block. Produces shadow log for comparison.

**replay** — re-executes from execution log using recorded values.
Deterministic. Non-perturbative. Refuses on schema mismatch.

---

## Part 9 — Examples

### Hello World

    fn hello(): Silence
      emit "Hello, World!"

    run hello()

### Bell state

    require quantum qubits: 2

    fn bell_state(): Distribution
      qubit a, b
      interfere a: hadamard
      let pair ← entangle a, b
      let result ← measure pair shots: 1000
      return result

    error CapabilityMissing(e):
      emit "QPU unavailable: " ++ e.message
      return Distribution.empty

    let dist ← run bell_state()
    ~ ~500x "00", ~500x "11", never "01" or "10"

### Quantum search with classical fallback

    fn search(data: [Text]): Result
      let has_qpu ← capability quantum
      let qubits  ← capability quantum.qubits

      has_qpu and qubits >= 8 ?
        qubit candidates ← superpose(data)
        interfere candidates: toward match(data)
        let result ← measure candidates shots: 1000
        return Result(
          answer:     result.most_probable,
          confidence: result.distribution.peak,
          shots:      1000
        )
      :
        let answer ← classical_search(data)
        return Result(answer: answer, confidence: 1.0, shots: 0)

### Thermal-aware execution

    require quantum qubits: 5
    require quantum fidelity: 0.999
    prefer thermal

    adapt thermal → priority
      backoff: exponential(min: 5.0, max: 60.0)
      suppress: oscillation:
      let temp ← measure thermal
      ~ wide hysteresis: 20mK gap prevents oscillation
      let priority ← temp.upper > 20.0 ? 0.1 :
                     temp.upper < 15.8 ? 0.9 : priority

    fn compute(input: [Frequency]): Distribution
      qubit q ← superpose(input)
      interfere q: toward solution(input)
      return measure q shots: 2000

### Mid-circuit measurement

    require quantum qubits: 2

    fn conditional_phase(data: [Text]): Collapsed
      qubit a, b
      interfere a: hadamard
      let pair ← entangle a, b
      let (flag, remaining) ← split pair measure: 0
      flag == 1 ?
        interfere remaining: phase_flip
      :
        interfere remaining: hadamard
      return measure remaining

---

## Build Path

The Python library comes first. Users and feedback before the compiler.

| Stage | What | Time |
|---|---|---|
| 1 | Python library: runtime ownership enforcement + log schema v1 | 2–4 weeks |
| 2 | Qiskit backend: Bell state on real IBM hardware | 1–2 weeks |
| 3 | Publish results. Gather feedback from quantum users. | ongoing |
| 4 | Grammar + parser + tree-walk interpreter | 1–2 months |
| 5 | Static type checker: qubit lifecycle and ownership | 1–2 months |
| 6 | OpenQASM 3.0 code generation | 1–2 months |
| 7 | Physical layer: capability, measure, adapt | 1 month |
| 8 | Simulate and replay debug modes | 1 month |
| 9 | Compiler to LLVM or WebAssembly | 6–18 months |

Stage 1 and 2 are provable in weeks on existing hardware.
Everything else builds on validated ideas from real users.

---

## Open Questions

1. Should `fn` support mutable closure capture from enclosing scope?

2. Should `measure` support streaming — continuous readings as a typed
   stream rather than discrete snapshots?

3. Should the compiler perform deeper oscillation analysis beyond the
   simple threshold pattern?

4. What is the right model for qubit reclamation when a parallel branch
   is halted mid-computation — before any qubit has been allocated but
   after the QPU job has been submitted?

---

## What Kelvin is

A quantum programming language. Linear types enforce physical reality.
The physical layer is the thesis, not an appendix. The compiler tells
you when you are wrong before the hardware does.

---

*Kelvin v0.7*
