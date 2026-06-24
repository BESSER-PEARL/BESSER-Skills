# BESSER Deployment Metamodel & Terraform Generator Reference

Full reference for building a `DeploymentModel` in Python and feeding it to
`TerraformGenerator`. Read this when the user is modelling clusters, nodes,
services, networking, or generating Terraform (GCP/AWS) infrastructure code.

## Table of contents

- [Imports](#imports)
- [Enumerations](#enumerations)
- [Building blocks](#building-blocks)
- [Clusters and the model root](#clusters-and-the-model-root)
- [Minimal runnable example](#minimal-runnable-example)
- [Generation](#generation)
- [Gotchas](#gotchas)

## Imports

Everything lives in one module and is re-exported from the package
(`from .deployment import *`):

```python
from besser.BUML.metamodel.deployment import (
    DeploymentModel,
    Cluster, PublicCluster, OnPremises,
    Node, CloudNode, EdgeNode,
    Application, Container, Deployment, Volume,
    Service, SecurityGroup,
    Network, Subnetwork, IPRange,
    Region, Zone,
    Resources,
    # Enums:
    Provider, ServiceType, Protocol, IPRangeType,
    Processor, Hypervisor,
)
from besser.generators.terraform import TerraformGenerator
```

`Application` takes a `DomainModel`, so you typically also import from
`besser.BUML.metamodel.structural`.

## Enumerations

All are plain `enum.Enum`; pass the **member** (e.g. `Provider.google`), not
the string.

| Enum | Members → value |
|------|-----------------|
| `Provider` | `google` → `"Google"`, `aws` → `"AWS"`, `azure` → `"Azure"`, `other` → `"Other"` |
| `ServiceType` | `lb` → `"LoadBalancer"`, `ingress` → `"Ingress"`, `egress` → `"Egress"` |
| `Protocol` | `http` → `"HTTP"`, `https` → `"HTTPS"`, `tcp` → `"TCP"`, `udp` → `"UDP"`, `all` → `"ALL"` |
| `IPRangeType` | `subnet` → `"Subnetwork"`, `pod` → `"Pod"`, `service` → `"Service"` |
| `Processor` | `x64` → `"x64"`, `x86` → `"x86"`, `arm` → `"ARM"` |
| `Hypervisor` | `vm_ware` → `"VMWare"`, `hyper_v` → `"Hyper-V"`, `xen_server` → `"XenServer"`, `rhev` → `"RHEV"`, `kvm` → `"KVM"` |

Only **`Provider.google`** and **`Provider.aws`** produce Terraform output;
any other provider is skipped at generation time (see Gotchas).

## Building blocks

`Resources` is a plain class; everything else extends `NamedElement`
(so the first positional arg is always `name`). Names follow B-UML naming
rules: no spaces, no hyphens.

```python
Resources(cpu: int, memory: int)          # memory in MB. NOT a NamedElement (no name)
```

| Class | Constructor signature (positional order) |
|-------|------------------------------------------|
| `Application` | `(name, image_repo, port, required_resources: Resources, domain_model: DomainModel)` |
| `Volume` | `(name, mount_path, sub_path)` |
| `Container` | `(name, application: Application, resources_limit: Resources = None, volumes: set[Volume] = None)` |
| `Deployment` | `(name, replicas: int, containers: set[Container])` |
| `Service` | `(name, port: int, target_port: int, type: ServiceType, protocol: Protocol, application: Application = None)` |
| `IPRange` | `(name, cidr_range: str, type: IPRangeType, public: bool)` |
| `SecurityGroup` | `(name, rules: set[Service])` |
| `Network` | `(name, security_groups: set[SecurityGroup] = None)` |
| `Subnetwork` | `(name, ip_ranges: set[IPRange], network: Network)` |
| `Zone` | `(name)` |
| `Region` | `(name, zones: set[Zone])` |
| `Node` | `(name, public_ip, private_ip, os, resources: Resources, storage: int, processor: Processor)` |
| `CloudNode` / `EdgeNode` | same signature as `Node` (subclasses, no extra args) |

Notes:
- `Container.volumes` defaults to an empty `set()`; `resources_limit`
  defaults to `None`.
- `Service.application` is optional. When a `Service` is used inside a
  `SecurityGroup.rules`, it acts as a firewall rule (port/protocol).

## Clusters and the model root

```python
Cluster(
    name, services: set[Service], deployments: set[Deployment],
    regions: set[Region], net_config: bool = True,
    nodes: set[Node] = None, networks: set[Network] = None,
    subnets: set[Subnetwork] = None,
)

PublicCluster(
    name, services: set[Service], deployments: set[Deployment],
    regions: set[Region], num_nodes: int, provider: Provider, config_file: str,
    networks: set[Network] = None, subnets: set[Subnetwork] = None,
    net_config: bool = True,
)   # extends Cluster — adds config_file, num_nodes, provider

OnPremises(
    name, services, deployments, regions, nodes, hypervisor: Hypervisor,
    networks, subnets,
)   # extends Cluster — see Gotchas (buggy super() call)

DeploymentModel(name, clusters: set[Cluster])
```

`TerraformGenerator` only consumes `PublicCluster` instances (it reads
`.config_file` and `.provider` on every cluster — see Gotchas).

## Minimal runnable example

This mirrors the upstream test fixture
(`tests/generators/terraform/test_terraform_generator.py`) and generates GCP
Terraform.

```python
from besser.BUML.metamodel.structural import DomainModel, Class, Property, StringType
from besser.BUML.metamodel.deployment import (
    DeploymentModel, PublicCluster, Deployment, Service, Container,
    Application, Resources, Region, Zone, ServiceType, Protocol, Provider,
)
from besser.generators.terraform import TerraformGenerator

# A domain model is required by Application
domain_model = DomainModel(
    name="TestDomain",
    types={Class(name="App", attributes={Property(name="name", type=StringType)})},
    associations=set(),
)

resources = Resources(cpu=1, memory=512)            # memory in MB
app = Application(
    name="test_app", image_repo="gcr.io/test/app", port=8080,
    required_resources=resources, domain_model=domain_model,
)

container = Container(name="app_container", application=app)
deployment = Deployment(name="app_deployment", replicas=2, containers={container})

service = Service(
    name="app_service", port=80, target_port=8080,
    type=ServiceType.lb, protocol=Protocol.http, application=app,
)

zone = Zone(name="us_central1_a")
region = Region(name="us_central1", zones={zone})

# A provider config file is REQUIRED and read at generation time.
# Lines are injected verbatim into the provider{} block of the .tf output.
with open("cluster.conf", "w") as f:
    f.write("project_id = test_project\n")
    f.write("cluster_name = test_cluster\n")

cluster = PublicCluster(
    name="test_cluster",
    services={service},
    deployments={deployment},
    regions={region},          # MUST be a set — templates iterate over it
    num_nodes=3,
    provider=Provider.google,
    config_file="cluster.conf",
)

model = DeploymentModel(name="TestDeployment", clusters={cluster})

TerraformGenerator(deployment_model=model, output_dir="./output").generate()
```

## Generation

`TerraformGenerator(deployment_model: DeploymentModel, output_dir: str = None)`

- Output goes to `output_dir`; if `None`, to `<cwd>/output`.
- `.generate()` iterates `deployment_model.clusters`. For each cluster it:
  1. opens `cluster.config_file` and reads its lines (skips the cluster with
     a printed error if the file can't be read),
  2. picks a template set from `cluster.provider.value` (`"Google"` or
     `"AWS"` only; otherwise prints `Unsupported provider` and skips),
  3. writes files into
     `<output_dir>/<provider_lower>_<cluster_name>/` (e.g.
     `google_test_cluster/`).
- GCP emits: `version.tf`, `cluster.tf`, `app.tf`, `api.tf`, `setup.bat`.
- AWS emits: `eks.tf`, `iam-oidc.tf`, `provider.tf`, `igw.tf`, `nat.tf`,
  `routes.tf`, `vpc.tf`, `nodes.tf`, `subnets.tf`, `setup.bat`.

There is no `validate()` step in this pipeline (unlike `DomainModel`).

## Gotchas

- **`config_file` is mandatory for `PublicCluster` and must exist on disk.**
  `generate()` opens it per cluster; its lines are dropped verbatim into the
  provider block of `version.tf`/`cluster.tf`/`api.tf` (GCP) /`provider.tf`
  (AWS). Use Terraform provider syntax, e.g. `project_id = "my-project"`.
- **`regions` must be a `set`** even though `PublicCluster`/`OnPremises`
  annotate it as `Region`. The templates do `for x in public_cluster.regions`,
  so a single `Region` object would iterate incorrectly. Pass `{region}`.
- **Only `Provider.google` and `Provider.aws` generate output.** `azure` and
  `other` hit the `else` branch (`get_template_to_file_map` returns `None`)
  and are silently skipped with a printed message.
- **The generator assumes every cluster is effectively a `PublicCluster`.** It
  unconditionally accesses `cluster.config_file` and `cluster.provider`, which
  plain `Cluster`/`OnPremises` instances do not have. Use `PublicCluster` for
  Terraform generation.
- **`OnPremises.__init__` has a parameter-order bug.** Its `super().__init__`
  passes `nodes` into the parent's `net_config` slot:
  `super().__init__(name, services, deployments, regions, nodes, networks, subnets)`
  while `Cluster.__init__(... net_config, nodes, networks, subnets)`. So for
  `OnPremises`, `net_config` ends up holding the nodes set and `nodes` ends up
  holding what you passed as `networks`, etc. `OnPremises` is not used by the
  Terraform generator; flag before relying on it.
- **GCP `app.tf.j2` reads `container.resources_limit.cpu`/`.memory`
  unconditionally.** `Container.resources_limit` defaults to `None`. Jinja2
  silently renders the missing attribute as empty (generation does not crash),
  but the `limits {}` block in the output `.tf` will be blank unless you pass a
  `Resources` to `resources_limit`. `required_resources` (on the Application)
  is always rendered as the `requests {}` block.
- **`Resources.memory` is in megabytes** and `Application.port` is typed `int`
  (the attribute setter stores whatever you pass; the test uses an `int`).
- These deployment classes are **not part of a domain model** and have **no
  `validate()`** of their own; errors surface only at generation time.
