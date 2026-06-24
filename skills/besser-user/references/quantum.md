# Quantum circuits and Qiskit generation

Reference for building a `QuantumCircuit` B-UML model in the BESSER quantum
metamodel and generating runnable Qiskit code with `QiskitGenerator`. Read
this when the user is modelling qubit registers, gates, and measurements, or
generating a Qiskit script.

## Table of contents

- [Imports](#imports)
- [Core model classes](#core-model-classes)
- [Gates](#gates)
- [Measurements and other operations](#measurements-and-other-operations)
- [Minimal build example](#minimal-build-example)
- [Generation](#generation)
- [Gotchas](#gotchas)

## Imports

Everything is re-exported from the package root (the package `__init__`
does `from .quantum import *`, and there is no `__all__`, so every class is
importable directly):

```python
from besser.BUML.metamodel.quantum import (
    QuantumCircuit, QuantumRegister, ClassicalRegister,
    Measurement, ControlState,
    # single-qubit gates
    HadamardGate, PauliXGate, PauliYGate, PauliZGate,
    SGate, TGate, IdentityGate, PhaseGate,
    RXGate, RYGate, RZGate,
    # multi-qubit gates
    CNOT, CXGate, CYGate, CZGate, CHGate, CPhaseGate,
    SwapGate, iSwapGate, SqrtSwapGate, BellGate,
    CRXGate, CRYGate, CRZGate,
    RXXGate, RYYGate, RZZGate, RZXGate,
    QFTGate,
)
from besser.generators.qiskit import QiskitGenerator
```

## Core model classes

| Class | Constructor | Notes |
|-------|-------------|-------|
| `QuantumCircuit` | `QuantumCircuit(name, qubits=0)` | Subclass of `Model`. If `qubits > 0`, a default quantum register named `"q"` of that size is auto-added. |
| `QuantumRegister` | `QuantumRegister(name, size)` | Builds `size` `Qubit`s indexed `0..size-1`. |
| `ClassicalRegister` | `ClassicalRegister(name, size)` | Builds `size` `ClassicalBit`s. |
| `ControlState` | enum: `ControlState.CONTROL`, `ControlState.ANTI_CONTROL` | `CONTROL` = active on \|1⟩ (→ `'1'`), `ANTI_CONTROL` = active on \|0⟩ (→ `'0'`). |

`QuantumCircuit` methods/properties:

- `add_qreg(qreg: QuantumRegister)` — append a quantum register.
- `add_creg(creg: ClassicalRegister)` — append a classical register.
- `add_operation(operation)` — append any gate / measurement / operation, in order.
- `num_qubits` (property) — sum of `size` over all `qregs`.
- `num_clbits` (property) — sum of `size` over all `cregs`.

State lives in the public lists `circuit.qregs`, `circuit.cregs`,
`circuit.operations`.

## Gates

All gate constructors take **integer qubit indices** (not `Qubit` objects),
addressed across the flattened circuit qubit space.

Single-qubit gates — signature `Gate(target_qubit: int, control_qubits=None, control_states=None)`:

- `HadamardGate`, `PauliXGate`, `PauliYGate`, `PauliZGate`, `SGate`, `TGate`
- `IdentityGate(target_qubit)` — **no** control params
- `PhaseGate(target_qubit, angle: float, control_qubits=None, control_states=None)`
- `RXGate` / `RYGate` / `RZGate` `(target_qubit, theta: float, control_qubits=None, control_states=None)` — rotation angle parameter is named **`theta`**

Adding `control_qubits=[...]` to any single-qubit gate turns it into a
controlled gate. By default each control is `ControlState.CONTROL`; pass
`control_states=[...]` (same length as `control_qubits`) to make any anti-controls.
A `PauliXGate` with one control is the standard way to express a CNOT.

Two-/multi-qubit gates (fixed positional args, **no** control params unless noted):

- `CNOT(control_qubit, target_qubit)` and its aliases `CXGate`, `CYGate`, `CZGate`, `CHGate`
- `CPhaseGate(control_qubit, target_qubit, theta: float)`
- `CRXGate` / `CRYGate` / `CRZGate` `(control_qubit, target_qubit, theta: float)`
- `SwapGate(qubit1, qubit2, control_qubits=None, control_states=None)` — the only multi-qubit gate that accepts control params
- `iSwapGate(qubit1, qubit2)`, `SqrtSwapGate(qubit1, qubit2)`, `BellGate(qubit1, qubit2)`
- `RXXGate` / `RYYGate` / `RZXGate` `(qubit1, qubit2, theta: float)`; `RZZGate(qubit1, qubit2, theta: float = None)`
- `QFTGate(target_qubits: List[int], inverse: bool = False)`

## Measurements and other operations

```python
Measurement(target_qubit: int, output_bit: int = None, basis: str = 'Z')
```

- `basis` is `'X'`, `'Y'`, or `'Z'` (default `'Z'`).
- If `output_bit` is `None`, the generator writes to the classical bit with
  the same index as `target_qubit`.

Other operation types exist in the metamodel and are handled by the
generator (`InputGate`, `CustomGate`, `FunctionGate`, `QFTGate`,
`PhaseGradientGate`, `OrderGate`, arithmetic/comparison gates,
`DisplayOperation`, `ScalarGate`, `PostSelection`, etc.). Most are aimed at
Quirk-style imports; many map to Qiskit placeholders. Stick to the standard
gates above for reliably runnable output.

## Minimal build example

```python
from besser.BUML.metamodel.quantum import (
    QuantumCircuit, ClassicalRegister,
    HadamardGate, PauliXGate, RXGate, Measurement, ControlState,
)
from besser.generators.qiskit import QiskitGenerator

# 3 qubits -> auto-creates a quantum register named "q"
circuit = QuantumCircuit("TestCircuit", qubits=3)
circuit.add_creg(ClassicalRegister("c", 3))   # classical register for results

circuit.add_operation(HadamardGate(target_qubit=0))
circuit.add_operation(PauliXGate(target_qubit=1, control_qubits=[0]))   # CNOT 0->1
circuit.add_operation(RXGate(target_qubit=2, theta=0.785))             # note: theta=
# anti-controlled gate:
circuit.add_operation(PauliXGate(
    target_qubit=1, control_qubits=[0],
    control_states=[ControlState.ANTI_CONTROL],
))

circuit.add_operation(Measurement(target_qubit=0, output_bit=0))
circuit.add_operation(Measurement(target_qubit=1, output_bit=1))
circuit.add_operation(Measurement(target_qubit=2, output_bit=2))

gen = QiskitGenerator(model=circuit, output_dir="./output")
gen.generate()   # writes ./output/qiskit_circuit.py
```

## Generation

```python
QiskitGenerator(
    model: QuantumCircuit,
    output_dir: str = None,
    backend_type: str = "aer_simulator",
    shots: int = 1024,
)
```

- Output file: always `qiskit_circuit.py` in `output_dir`. If `output_dir`
  is `None`, it writes to an `output/` folder under the current working
  directory.
- `backend_type` must be one of `"aer_simulator"`, `"fake_backend"`,
  `"ibm_quantum"` (held in `VALID_BACKENDS`). Any other value raises
  `ValueError("Invalid backend: ...")` at construction time.
- `shots` (default `1024`) is inlined into the generated execution code.

What each backend emits in the generated script:

| `backend_type` | Imports / execution in generated code |
|----------------|----------------------------------------|
| `aer_simulator` | `from qiskit_aer import AerSimulator`; runs locally via `AerSimulator()` + `transpile` + `.run(..., shots=...)`. |
| `fake_backend` | Uses `FakeManilaV2` from `qiskit_ibm_runtime.fake_provider` with `AerSimulator.from_backend(...)` to model real-hardware noise. |
| `ibm_quantum` | Uses `QiskitRuntimeService` + `SamplerV2`; picks the least-busy real backend (requires saved IBM Quantum credentials). |

If the model has any `Measurement` but no classical registers, the generator
auto-adds a classical register named `"c"` sized to `num_qubits` before
rendering. The generated script always ends by printing `qc.draw()` and the
measurement `counts`.

## Gotchas

- **Rotation parameter is `theta`, not `angle`** — `RXGate`, `RYGate`,
  `RZGate`, and the controlled/two-qubit rotation gates all use `theta`.
  (Only `PhaseGate` uses `angle`, which it stores as `self.parameter`.) Using
  `angle=` on an `RXGate` raises `TypeError`.
- Qubit references are plain `int` indices, not `Qubit` objects.
- `QuantumCircuit(name, qubits=N)` with `N>0` already adds the `"q"` register;
  do not also `add_qreg` a second `"q"` or you double the qubit count.
- `control_states`, when given, must match the length of `control_qubits`.
  Omit it to default every control to `CONTROL`.
- The generator targets modern Qiskit (`qiskit_aer`, `qiskit_ibm_runtime`,
  `SamplerV2`); the generated file requires those packages installed to run.
- Non-standard operations (custom/arithmetic/comparison/display gates) often
  render as opaque `create_placeholder(...)` barriers — generated but not
  semantically meaningful. Prefer the listed standard gates for executable output.
