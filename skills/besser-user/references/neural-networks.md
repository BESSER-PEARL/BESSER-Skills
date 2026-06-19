# BESSER Neural Network Models Reference

Build a neural network as a B-UML model (layers + tensor ops + training
configuration + datasets), then generate runnable PyTorch or TensorFlow
training/evaluation code from it. The metamodel lives in
`besser.BUML.metamodel.nn`; the generators in
`besser.generators.nn.{pytorch,tf}`.

Verified against BESSER v7.8.3.

## Table of contents

- [Imports](#imports)
- [Core classes](#core-classes)
- [Layers](#layers)
- [Tensor operations](#tensor-operations)
- [Activation functions](#activation-functions)
- [Datasets and configuration](#datasets-and-configuration)
- [Minimal build example](#minimal-build-example)
- [Generation](#generation)
- [Sub-networks (composition)](#sub-networks-composition)
- [Validation](#validation)
- [Gotchas](#gotchas)

## Imports

The metamodel `__init__` re-exports everything via `from .neural_network
import *`, so import names directly from `besser.BUML.metamodel.nn`:

```python
from besser.BUML.metamodel.nn import (
    NN, Configuration, Dataset, Image, Label, Feature, Structured,
    Layer, TensorOp,
    # convolution / pooling
    Conv1D, Conv2D, Conv3D, PoolingLayer,
    # normalization / dropout
    BatchNormLayer, LayerNormLayer, DropoutLayer,
    # recurrent
    SimpleRNNLayer, LSTMLayer, GRULayer,
    # general
    LinearLayer, FlattenLayer, EmbeddingLayer,
)
```

The generator package `__init__.py` files have their re-exports commented
out, so import the generator classes from their full module path (not the
package):

```python
from besser.generators.nn.pytorch.pytorch_code_generator import PytorchGenerator
from besser.generators.nn.tf.tf_code_generator import TFGenerator
```

## Core classes

| Class | One-liner |
|-------|-----------|
| `NN(name, configuration=None, train_data=None, test_data=None)` | The model. Subclass of `BehaviorImplementation`. Holds an ordered list of modules (layers, tensor ops, sub-NNs). |
| `Configuration(batch_size, epochs, learning_rate, optimizer, loss_function, metrics, weight_decay=0, momentum=0)` | Training/eval hyperparameters. `metrics` is a `List[str]`. |
| `Dataset(name, path_data, task_type=None, input_format=None, image=None, labels=None)` | Points at training/test data on disk. |
| `Image(shape=None, normalize=False)` | Image-feature descriptor; `shape` defaults to `[256, 256]`. |
| `Label(col_name, label_name=None)` | A target label / column. |

`NN` is assembled with **chainable** `add_*` methods (each returns `self`):
`add_layer(layer)`, `add_tensor_op(tensor_op)`, `add_sub_nn(sub_nn)`,
`add_configuration(cfg)`, `add_train_data(ds)`, `add_test_data(ds)`.
The collections `nn.layers`, `nn.tensor_ops`, `nn.sub_nns`, and `nn.modules`
are read-only — use the `add_*` methods; assigning to them raises
`AttributeError`. Module **order matters**: it defines the forward sequence.

## Layers

All layers descend from `Layer(name, actv_func=None, name_module_input=None,
input_reused=False)`. Common kwargs:
- `actv_func` (str | None): activation applied after the layer (see below).
- `name_module_input` (str): name of the module whose output feeds this layer
  (defaults to the previous module in sequence — set this only to wire a
  non-sequential input).
- `input_reused` (bool): the input to this layer is also fed to another layer.

**Convolution** (`ConvolutionalLayer` subclasses; note `out_channels` is
required, `stride_dim`/`in_channels` are optional):

```python
Conv1D(name, kernel_dim, out_channels, stride_dim=None, in_channels=None,
       padding_amount=0, padding_type="valid", actv_func=None, ...)
Conv2D(name, kernel_dim, out_channels, stride_dim=None, in_channels=None, ...)
Conv3D(name, kernel_dim, out_channels, stride_dim=None, in_channels=None, ...)
```
- `kernel_dim` must have exactly 1 / 2 / 3 elements for Conv1D/2D/3D (validated).
- `stride_dim` defaults to `[1]`, `[1,1]`, `[1,1,1]` respectively.
- `padding_type` must be `"same"` or `"valid"` (validated; default `"valid"`).
- `permute_in` / `permute_out` (bool): PyTorch-only, permute conv input/output
  to match the TF channel convention.

**Pooling**:

```python
PoolingLayer(name, pooling_type, dimension, kernel_dim=None, stride_dim=None,
             padding_amount=0, padding_type="valid", output_dim=None, ...)
```
- `pooling_type` ∈ `{'average', 'adaptive_average', 'max', 'adaptive_max',
  'global_average', 'global_max'}` (validated).
- `dimension` ∈ `{'1D', '2D', '3D'}` (validated).
- For non-adaptive/non-global pooling, `kernel_dim` length must match
  `dimension`. `output_dim` is only relevant for adaptive pooling.

**Normalization / dropout**:

```python
BatchNormLayer(name, num_features, dimension, actv_func=None, ...)   # dimension ∈ '1D'/'2D'/'3D'
LayerNormLayer(name, normalized_shape, actv_func=None, ...)          # normalized_shape: List[int]
DropoutLayer(name, rate, name_module_input=None, input_reused=False) # rate: float in [0, 1)
```
`DropoutLayer` takes no `actv_func`.

**Recurrent** (all subclass `RNN`; `hidden_size` required):

```python
SimpleRNNLayer(name, hidden_size, return_type="full", input_size=None,
               bidirectional=False, dropout=0.0, batch_first=True, actv_func=None, ...)
LSTMLayer(name, hidden_size, return_type="full", input_size=None, ...)
GRULayer(name, hidden_size, return_type="full", input_size=None, ...)
```
- `return_type` ∈ `{'hidden', 'last', 'full'}` (validated; default `'full'`).
- `batch_first` defaults to `True` (PyTorch-only flag). `dropout` defaults `0.0`.

**General**:

```python
LinearLayer(name, out_features, in_features=None, actv_func=None, ...)   # dense / fully-connected
FlattenLayer(name, start_dim=1, end_dim=-1, actv_func=None, ...)
EmbeddingLayer(name, num_embeddings, embedding_dim, actv_func=None, ...)
```

> `in_channels` / `in_features` / `input_size` are optional — when omitted the
> generators infer them where possible. Provide them when in doubt.

## Tensor operations

```python
TensorOp(name, tns_type, concatenate_dim=None, layers_of_tensors=None,
         reshape_dim=None, transpose_dim=None, permute_dim=None,
         input_reused=False)
```

The first positional name argument is `name`; the type argument keyword is
**`tns_type`** (not `type`). `tns_type` ∈ `{'reshape', 'concatenate',
'multiply', 'matmultiply', 'permute', 'transpose'}` (validated). Each type
requires its companion parameter, or construction raises `ValueError`:

| `tns_type` | required parameter |
|------------|--------------------|
| `reshape` | `reshape_dim: List[int]` |
| `concatenate` | `concatenate_dim: int` **and** `layers_of_tensors: List` |
| `multiply` | `layers_of_tensors` |
| `matmultiply` | `layers_of_tensors` |
| `permute` | `permute_dim: List[int]` |
| `transpose` | `transpose_dim: List[int]` |

`layers_of_tensors` entries are layer names (str) or scalar values (float).

```python
nn_model.add_tensor_op(TensorOp(name="op1", tns_type="concatenate",
                                layers_of_tensors=["l4", "l6"],
                                concatenate_dim=-1))
```

## Activation functions

`actv_func` is **not** validated by the metamodel (the validator is commented
out), but the code generators only translate this set:
`'relu'`, `'leaky_relu'`, `'sigmoid'`, `'softmax'`, `'tanh'`.
Use `None` (or omit) for no activation. Other strings will not generate a
matching activation.

## Datasets and configuration

```python
Configuration(batch_size, epochs, learning_rate, optimizer, loss_function,
              metrics, weight_decay=0, momentum=0)
```
- `optimizer` ∈ `{'sgd', 'adam', 'adamW', 'adagrad'}` (validated).
- `loss_function` ∈ `{'crossentropy', 'binary_crossentropy', 'mse'}` (validated).
- `metrics` is a **list** of strings, each ∈
  `{'accuracy', 'precision', 'recall', 'f1-score', 'mae'}` (validated).

```python
Dataset(name, path_data, task_type=None, input_format=None, image=None, labels=None)
```
- `task_type` ∈ `{'binary', 'multi_class', 'regression'}` (validated when set).
- `input_format` ∈ `{'csv', 'images'}` (validated when set).
- `image=Image(shape=[H, W, C])` when `input_format='images'`.

## Minimal build example

```python
from besser.BUML.metamodel.nn import (
    NN, Conv2D, PoolingLayer, FlattenLayer, LinearLayer,
    Configuration, Image, Dataset,
)
from besser.generators.nn.pytorch.pytorch_code_generator import PytorchGenerator
from besser.generators.nn.tf.tf_code_generator import TFGenerator

nn_model = NN(name="my_model")
nn_model.add_layer(Conv2D(name="l1", actv_func="relu", in_channels=3,
                          out_channels=32, kernel_dim=[3, 3]))
nn_model.add_layer(PoolingLayer(name="l2", pooling_type="max",
                                dimension="2D", kernel_dim=[2, 2]))
nn_model.add_layer(Conv2D(name="l3", actv_func="relu", in_channels=32,
                          out_channels=64, kernel_dim=[3, 3]))
nn_model.add_layer(FlattenLayer(name="l4"))
nn_model.add_layer(LinearLayer(name="l5", actv_func="relu",
                               in_features=1024, out_features=64))
nn_model.add_layer(LinearLayer(name="l6", in_features=64, out_features=10))

nn_model.add_configuration(Configuration(
    batch_size=32, epochs=10, learning_rate=0.001,
    optimizer="adam", loss_function="crossentropy", metrics=["f1-score"]))

image = Image(shape=[32, 32, 3], normalize=False)
nn_model.add_train_data(Dataset(name="train_data", path_data="dataset/train",
                                task_type="multi_class", input_format="images",
                                image=image))
nn_model.add_test_data(Dataset(name="test_data", path_data="dataset/test"))

PytorchGenerator(model=nn_model, output_dir="output").generate()
TFGenerator(model=nn_model, output_dir="output").generate()
```

## Generation

```python
PytorchGenerator(model, output_dir=None, generation_type="subclassing",
                 channel_last=False)
TFGenerator(model, output_dir=None, generation_type="subclassing")
```

- `generation_type` is a **constructor argument** (not passed to `generate()`):
  `"subclassing"` (default) or `"sequential"`.
- `channel_last` is **PyTorch only** — when `True`, conv layers' input/output
  are permuted to match the TF channel-last convention. `TFGenerator` has no
  such parameter.
- `generate()` takes no required arguments. It renders a Jinja template and
  writes the file, printing the path. If `output_dir` is `None`, output goes to
  `<current dir>/output`.
- Output filename is derived from the fixed base name plus the generation type:
  PyTorch → `pytorch_nn_<generation_type>.py`,
  TF → `tf_nn_<generation_type>.py`
  (e.g. `pytorch_nn_subclassing.py`).

## Sub-networks (composition)

An `NN` can contain other `NN`s as modules (e.g. a `features` block and a
`classifier` block), added in order with `add_sub_nn`:

```python
features = NN(name="features")
features.add_layer(Conv2D(name="c1", in_channels=3, out_channels=64,
                          kernel_dim=[3, 3], actv_func="relu"))

classifier = NN(name="classifier")
classifier.add_layer(LinearLayer(name="fc", in_features=64, out_features=10))

model = NN(name="alexnet")
model.add_sub_nn(features)
model.add_sub_nn(classifier)
```

## Validation

`NN.validate(raise_exception=True)` returns
`{"success": bool, "errors": [...], "warnings": [...]}`. It checks: unique
module names within an NN; `name_module_input` references resolve to a module
in the same NN; `TensorOp.layers_of_tensors` string entries resolve; the first
module is an entry point (no declared input); sub-NN graph is acyclic; names are
valid Python identifiers; and numerical bounds (positive sizes/rates, dropout in
`[0,1)`, etc.). It emits warnings for empty NNs and for training data without a
configuration. With `raise_exception=True` (default), errors raise `ValueError`.

## Gotchas

- **Import generators from the full module path**, not the package. The
  `besser/generators/nn/{pytorch,tf}/__init__.py` re-exports are commented out,
  so `from besser.generators.nn.pytorch import PytorchGenerator` will fail.
- **`TensorOp` uses `tns_type=`, not `type=`.** Construction validates that the
  matching dimension/list parameter is present, raising `ValueError` otherwise.
- **`generation_type` goes in the constructor**, not `generate()`.
- **`channel_last` is PyTorch-only.** Passing it to `TFGenerator` raises a
  `TypeError`.
- **Module order is the forward order.** The first module must not set
  `name_module_input`. Use `name_module_input` / `input_reused` for skip or
  branched connections (e.g. the `concatenate` TensorOp pattern).
- **`metrics` must be a list** even for a single metric: `metrics=["accuracy"]`.
- **Validated enums raise on bad values** at construction time: `optimizer`,
  `loss_function`, `metrics`, `task_type`, `input_format`, `padding_type`,
  `pooling_type`, `dimension`, `return_type`, `tns_type`.
- **`actv_func` is NOT validated** by the metamodel, but only
  `relu / leaky_relu / sigmoid / softmax / tanh` are translated by the
  generators; anything else is silently ignored.
```
