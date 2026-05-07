# Agent and specialty generators

Reference for generators that target chatbots, quantum circuits, neural
networks, or infrastructure. Read this when the user is running
`BAFGenerator`, `QiskitGenerator`, `TerraformGenerator`,
`PytorchGenerator`, or `TFGenerator`.

## BAFGenerator (chatbots)

```python
from besser.generators.agents import BAFGenerator
gen = BAFGenerator(model=agent, output_dir="./agent_output")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `Agent` (from `besser.BUML.metamodel.state_machine.agent`) |
| Output | `{agent_name}.py`, `config.ini`, `readme.txt`, optional RAG subdirectories |
| Options | `generation_mode` (`FULL`, `PERSONALIZED_ONLY`, `CODE_ONLY`), `config_path`/`config` for personalization, `openai_api_key` for LLM personalization |
| Known limits | Telegram handler is not fully implemented. Platform names are hardcoded in the template. |

For agent modeling itself (states, intents, training sentences,
transitions), see the **besser-user** skill's `references/agents.md`.

## QiskitGenerator (quantum)

```python
from besser.generators.qiskit import QiskitGenerator
gen = QiskitGenerator(model=circuit, output_dir="./output")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `QuantumCircuit` (from `besser.BUML.metamodel.quantum`) |
| Output | Single file: `qiskit_circuit.py` |
| Options | `backend_type` (`aer_simulator`, `fake_backend`, `ibm_quantum`), `shots` (default 1024) |
| Error | `ValueError` if `backend_type` is invalid. |
| Note | `qiskit` is required to *run* the generated code, not to generate it. |

## TerraformGenerator (infrastructure)

```python
from besser.generators.terraform import TerraformGenerator
gen = TerraformGenerator(model=deployment_model, output_dir="./infra")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DeploymentModel` |
| Output | `.tf` files per cluster |
| Providers | GCP and AWS supported. |

## PytorchGenerator / TFGenerator (neural networks)

```python
from besser.generators.pytorch import PytorchGenerator
gen = PytorchGenerator(model=nn_model, output_dir="./output")
gen.generate()                                 # default: subclassing API
gen.generate(generation_type="sequential")     # alternative: nn.Sequential
```

| Aspect | Detail |
|--------|--------|
| Input | NN model (`besser.BUML.metamodel.nn`) |
| Output | A PyTorch (or TensorFlow) script |
| Options | `generation_type`: `"subclassing"` or `"sequential"` |

`TFGenerator` works the same way for TensorFlow targets.
