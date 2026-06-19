# Feature Model Reference (B-UML)

Build feature models (software product line variability) in Python with B-UML.
A feature model is a hierarchy of `Feature` nodes connected by typed
`FeatureGroup` edges (mandatory / optional / alternative / or). Read this when
the user is modeling features, cardinality, feature groups, or feature
configurations.

## Imports

```python
from besser.BUML.metamodel.feature_model.feature_model import (
    FeatureModel, Feature, FeatureGroup, FeatureValue, FeatureConfiguration,
    MANDATORY, OPTIONAL, ALTERNATIVE, OR,   # group-kind string constants
)
```

The package `__init__.py` does `from .feature_model import *`, so
`from besser.BUML.metamodel.feature_model import FeatureModel, Feature, ...`
also works.

## Naming rules

`Feature` and `FeatureModel` are `NamedElement`s, so names must not contain
spaces or hyphens (use `_`). `FeatureGroup`, `FeatureValue`, and
`FeatureConfiguration` are plain `Element`s (no `name`).

## Key classes

| Class | Base | Purpose |
|-------|------|---------|
| `FeatureModel(name)` | `Model` | Root container; holds one `root_feature`. |
| `Feature(name, min=1, max=1, value=None)` | `NamedElement` | A feature node with cardinality and an optional `FeatureValue`. |
| `FeatureGroup(kind, features=None)` | `Element` | A typed edge from a parent to one or more child features. |
| `FeatureValue(t, values=None, min=None, max=None)` | `Element` | A typed attribute attached to a feature, configured later. |
| `FeatureConfiguration(feature, value=None)` | `Element` | A node in a concrete product configuration tree. |

### Group kinds (`FeatureGroup.kind`)

The four valid kinds are module-level string constants:

| Constant | Value | Children allowed | Meaning |
|----------|---------|------------------|---------|
| `MANDATORY` | `'mandatory'` | exactly 1 | child must be selected |
| `OPTIONAL` | `'optional'` | exactly 1 | child may be selected |
| `ALTERNATIVE` | `'alternative'` | 2 or more | pick exactly one (XOR) |
| `OR` | `'or'` | 2 or more | pick one or more |

`FeatureGroup.__init__` enforces this: `mandatory`/`optional` raise
`ValueError` if given more than 1 feature; `alternative`/`or` raise if given
fewer than 2.

### Feature cardinality

`Feature(name, min, max)` defaults to `min=1, max=1`. The constructor raises
`ValueError('Error in <name>: 0 < min < max')` if `min > max` or `min < 1`.
(So `min` must be at least 1 — there is no `min=0`; use an `optional` group for
"may be absent".)

### Building the hierarchy (fluent API)

`Feature` exposes chainable builder methods that create the child
`FeatureGroup` and set the child's `parent`. Each returns the **parent**
feature, so calls chain:

- `f.mandatory(child)` -> `Feature`
- `f.optional(child)` -> `Feature`
- `f.alternative([c1, c2, ...])` -> `Feature`
- `f.or_([c1, c2, ...])` -> `Feature`  (note the trailing underscore — `or` is a keyword)

Each method raises `ValueError` if a child already has a parent (a feature can
belong to only one parent). `FeatureModel.root(feature)` sets `root_feature`
and returns the model (also chainable).

## Minimal runnable example

```python
from besser.BUML.metamodel.feature_model.feature_model import (
    FeatureModel, Feature, FeatureValue,
)

feature_model = FeatureModel('feature_model').root(
    Feature('feature1')
    .mandatory(Feature('feature2'))
    .optional(Feature('feature2b'))
    .alternative([
        Feature('feature3'),
        Feature('feature4'),
    ])
    .or_([
        Feature('feature5'),
        Feature('feature6')
        .mandatory(
            Feature('feature7', min=2, max=10,
                    value=FeatureValue('int', values=[1, 2, 3]))
        ),
    ])
)

print(feature_model.root_feature.to_json())
print(feature_model.root_feature.get_depth())  # max nesting depth
```

## Feature attributes (`FeatureValue`)

`FeatureValue(t, values=None, min=None, max=None)` attaches a typed value to a
feature. `t` is one of `'int'`, `'float'`, `'str'`.

- Pass **either** an explicit `values` list **or** a `min`/`max` range — not
  both, and not neither. Mixing them (e.g. `values=[1,2,3], min=1`) or passing
  none raises `ValueError('Invalid arguments')`.
- When `values` is given, every element must match `t`, else `ValueError`
  (`'Value must be an integer'` / `' Value must be a string'` / `'Value must be a float'`).
  Note: `float` checking uses `isinstance(x, float)`, so integer literals like
  `1` are rejected for `t='float'` — use `1.0`.

## Configurations (`FeatureConfiguration`)

A concrete product is a separate tree of `FeatureConfiguration` nodes (it is
**not** auto-derived or validated against the feature model):

```python
from besser.BUML.metamodel.feature_model.feature_model import FeatureConfiguration

root_cfg = FeatureConfiguration(feature1)              # feature1 is a Feature
child_cfg = FeatureConfiguration(feature7, value=2)    # value fills a FeatureValue
root_cfg.add_child(child_cfg)
# or root_cfg.add_children([cfg_a, cfg_b])

root_cfg.get_child('feature7')      # single child by feature name, or None
root_cfg.get_children('feature7')   # list of children by feature name
root_cfg.get_depth()                # max depth
root_cfg.to_json()
```

`get_child` raises `ValueError` if more than one child shares the requested
name.

## Modularity / reuse

- `FeatureModel.duplicate(min=1, max=1) -> Feature` deep-copies the model's
  `root_feature` (and its groups) so the same sub-tree can be attached
  elsewhere. `Feature.duplicate(f, parent=None, min=1, max=1)` is the static
  helper it delegates to.
- This enables attaching one feature model under another for reusability.

## Notation / generation

There is **no dedicated code generator** for feature models in BESSER. The only
serialization is `Feature.to_json()` / `FeatureConfiguration.to_json()`, which
return nested `dict`s. `Feature.to_json()` emits
`{name: [{'kind': ..., 'features': [...]}, ...]}`; configurations emit
`{name: value}` for leaf attributes or nested dicts/lists for children.

## Gotchas

- Use `or_` (trailing underscore), not `or`.
- `min` cannot be 0; for "may be omitted" use an `optional` group, not
  cardinality.
- `FeatureValue`: supply `values` XOR (`min`/`max`); supplying both or neither
  raises `Invalid arguments`. `float` values must be real floats (`1.0`, not `1`).
- Builder methods (`mandatory`/`optional`/`alternative`/`or_`) and `root`
  return the parent/model for chaining; the child-group object is stored in
  `parent.children_groups`.
- A `Feature` can have only one parent — re-attaching a feature already in the
  tree raises `ValueError`.
- The model performs **no global/semantic validation** (no `validate()` method,
  no check that a `FeatureConfiguration` respects the model's
  constraints/cardinalities). Validation is limited to the per-constructor
  `ValueError`s described above.
- Equality (`__eq__`) is defined structurally on every class (by name +
  cardinality + value + children for `Feature`, etc.), which is what makes
  `feature_model1 == feature_model2` work after `duplicate()`.
