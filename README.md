# Swarm Cloud Marketplace Specification (v1alpha1)

> Declarative specifications for the Swarm Cloud Marketplace: deployable multi-chart applications (`AppDefinition`), published datasets (`DataDefinition`), and the versioned data-connection contracts that link them (`DataInterface`).

## Table of Contents

- [Overview](#overview)
- [Design Principles](#design-principles)
- [Repository Structure](#repository-structure)
- [App Definition Specification](#app-definition-specification)
  - [1. Metadata](#1-metadata)
  - [2. Components](#2-components)
    - [2.1 Component Structure](#21-component-structure)
    - [2.2 Source](#22-source)
    - [2.3 Deployment](#23-deployment)
    - [2.4 Patch Operations](#24-patch-operations)
    - [2.5 Evaluation Order](#25-evaluation-order)
    - [2.6 Dependencies & Deploy Order](#26-dependencies--deploy-order)
    - [2.7 Images](#27-images)
    - [2.8 Deployment Evidence](#28-deployment-evidence)
  - [3. UI / Presentation](#3-ui--presentation)
  - [4. Parameters](#4-parameters)
  - [5. Options / Value Source](#5-options--value-source)
  - [6. Resource Requirements](#6-resource-requirements)
  - [7. Expressions](#7-expressions)
  - [8. Outputs](#8-outputs)
  - [9. Data Slots](#9-data-slots)
- [Data Interface Specification](#data-interface-specification)
- [Data Definition Specification](#data-definition-specification)
- [Publisher Secrets](#publisher-secrets)
- [Compatibility & Binding](#compatibility--binding)
- [Versioning](#versioning)
- [Full Examples](#full-examples)
- [JSON Schemas](#json-schemas)

---

## Overview

The Swarm Cloud Marketplace is a Git-based catalog for a decentralized cloud platform. It hosts two kinds of listings -- **applications** and **datasets** -- linked by a registry of **data interfaces**. Everything is defined declaratively in YAML; applications are deployed into Kubernetes cluster spaces via Helm.

This specification defines three document kinds:

| Kind             | Location                            | Purpose                                                                                    |
|------------------|-------------------------------------|--------------------------------------------------------------------------------------------|
| `AppDefinition`  | `apps/<name>/app.yaml`              | A deployable application: components, parameters, UI, resource requirements, data slots.   |
| `DataInterface`  | `interfaces/<name>/<version>.yaml`  | A versioned contract for one way of connecting to data (the **protocol layer**).           |
| `DataDefinition` | `datasets/<name>/data.yaml`         | A published dataset: provided interfaces and semantic schema.                              |

For applications, the specification defines the **contract** between application authors and the platform:

- What the application is (metadata)
- What the user can configure (parameters, UI)
- What resources the application needs (resource requirements)
- What components make up the application (components with source and deployment)
- How user input maps to Helm releases (deployment values per component)
- What external data the application can consume (data slots)

An application may consist of one or more **components**. Each component corresponds to a single Helm chart. Multi-chart applications define multiple components in the `components` array, with optional dependency declarations to control deploy order.

For data, the specification defines how datasets declare what they contain (the **semantic layer**) and how they can be reached (the **protocol layer**), and how the platform decides which datasets are compatible with which applications -- see [Compatibility & Binding](#compatibility--binding).

**Scope:** This specification covers the marketplace repository layer only. Runtime behavior, operator logic, and cluster orchestration are separate concerns.

---

## Design Principles

### Parameters ≠ Helm Values

Parameters are a **product-level abstraction**. They represent meaningful choices a user makes (e.g., "which model to run", "enable public access"). Helm values are an **implementation detail** -- the low-level configuration passed to a Helm chart.

The mapping between the two is defined explicitly in each component's [Deployment](#23-deployment) section via `base` values and `patches`. This separation means:

- The user-facing form can change independently of the Helm charts.
- The same parameter can affect multiple Helm values across multiple components.
- Helm values that are not user-configurable are set in `base` and never exposed.

### Helm Is an Implementation Detail

Helm is used **only as a rendering and packaging engine**. The AppDefinition does not assume Helm-specific behavior beyond template rendering. The platform may support additional backends in the future without changing the parameter model.

### AppDefinition Is a Product Contract

An `app.yaml` file is a product specification, not a DevOps artifact. It defines what the user sees, what they can configure, and what guarantees the platform makes. It must be readable by frontend engineers building the deployment UI, backend engineers implementing the deployment pipeline, and platform engineers making scheduling decisions.

### Static to Dynamic Evolution

Option lists (e.g., available models) are currently defined statically in the app definition. The specification includes a `source` abstraction designed to support remote/dynamic data sources in the future without breaking existing definitions.

### Reproducibility

Given the same set of parameter values and the same app definition version, the platform MUST produce identical Helm values for every component. This ensures deployments are deterministic and auditable. The same holds for data: given the same slot selections, interface negotiation and compatibility matching MUST resolve identically. With container images pinned by digest ([Images](#27-images)), determinism extends to the rendered manifests themselves -- the expected deployment evidence digest is computable from the definition before anything is deployed (see [Deployment Evidence](#28-deployment-evidence)).

### Interfaces Are First-Class Contracts

Applications and datasets never reference each other directly. Both reference **DataInterfaces** -- named, versioned connection contracts maintained in the marketplace registry. Compatibility between an app and a dataset is computed from their declarations alone, before anything is deployed. Adding a new interface is a pull request; existing definitions are never touched.

### Data Access Is Brokered, Never Embedded

A `DataDefinition` never contains secret values or access rules. Connection configuration references [publisher secrets](#publisher-secrets) by name; the values live in the owner's console vault and are sealed for the TEEs that consume them. Connections are resolved into a **binding** at deploy time -- statically, or by a credential provisioner that mints individual credentials per consumer ([Connection Resolution](#connection-resolution)). Secret connection properties are released only after the deployment's evidence has been verified. Neither the app author, nor the deploying user, nor the marketplace operator ever sees dataset credentials. Visibility, usage terms, and grants are configured on the marketplace listing -- see [Secrets & Attestation](#secrets--attestation).

### Definitions Describe Facts, Listings Describe Policy

Source-stored definitions capture what is **intrinsic and slow-moving**: what an app deploys, what a dataset contains, how it can be reached. Everything that is a business decision of the publisher -- visibility, pricing, usage terms, per-organization and per-application grants, request-and-approval -- lives on the marketplace listing and can change at any moment without republishing. As a rule of thumb: if changing it should not produce a new version in Git history, it does not belong in a definition.

---

## Repository Structure

```
apps/
  <app-name>/
    app.yaml              # Application definition (required)
    README.md             # Human-readable documentation (required)
    icon.svg              # Application icon (optional)

interfaces/
  <interface-name>/
    <version>.yaml        # DataInterface contract, one file per version

datasets/
  <dataset-name>/
    data.yaml             # Dataset definition (required)
    README.md             # Human-readable documentation (required)

specs/
  app-definition.schema.json
  component.schema.json
  parameter.schema.json
  value-source.schema.json
  deployment.schema.json
  resource-requirements.schema.json
  data-interface.schema.json
  data-definition.schema.json
  data-schema.schema.json
  data-slot.schema.json
```

### `apps/<app-name>/app.yaml`

The primary application definition file. Must conform to the AppDefinition schema. The `<app-name>` directory name MUST match the `metadata.name` field in `app.yaml`.

### `apps/<app-name>/README.md`

Human-readable documentation for the application. Should describe what the app does, any prerequisites, and usage notes.

### `apps/<app-name>/icon.svg`

Optional SVG icon displayed in the marketplace UI. Recommended size: 64x64 logical pixels.

### `interfaces/<interface-name>/<version>.yaml`

A [DataInterface](#data-interface-specification) contract. One file per contract version; versions coexist. New interfaces and new versions are added via pull request.

### `datasets/<dataset-name>/data.yaml`

A [DataDefinition](#data-definition-specification) describing a published dataset. Datasets published from the organization admin console are represented by platform-generated DataDefinitions; this directory hosts curated public definitions. The contract is identical either way.

### `datasets/<dataset-name>/README.md`

Human-readable documentation for the dataset.

### `specs/`

JSON Schema files that formally define the structure of all entities in this specification. Used for validation and code generation.

---

## App Definition Specification

Every `app.yaml` starts with a top-level structure:

```yaml
apiVersion: swarm.cloud/v1alpha1
kind: AppDefinition

metadata: { ... }
ui: { ... }
parameters: [ ... ]
secrets: [ ... ]      # optional -- publisher secrets, see Publisher Secrets
resources: { ... }
data:                 # optional -- data slots, see section 9
  slots: [ ... ]
components:
  - name: <component-name>
    source: { ... }
    deployment: { ... }
  - name: <another-component>
    dependsOn: [<component-name>]
    source: { ... }
    deployment: { ... }
outputs: [ ... ]      # optional
```

---

### 1. Metadata

Describes the application for catalog display and search.

| Field         | Type       | Required | Description                                      |
|---------------|------------|----------|--------------------------------------------------|
| `name`        | `string`   | Yes      | Unique identifier. Must match `[a-z0-9-]+`.      |
| `title`       | `string`   | Yes      | Human-readable display name.                     |
| `description` | `string`   | Yes      | Short description (1-2 sentences).               |
| `version`     | `string`   | No       | User-facing application version (SemVer), independent of component chart versions. Shown in the listing's version history. |
| `category`    | `string`   | No       | Single catalog category from the marketplace taxonomy (e.g. `ai-inference`, `data-storage`, `web-apps`). Tags remain free-form. |
| `sourceRepo`  | `string`   | No       | URL of the application's source repository. Input for audit agencies and AI review. |
| `license`     | `string`   | No       | SPDX license identifier of the application's source code (e.g. `MIT`, `Apache-2.0`). |
| `icon`        | `string`   | No       | Relative path to the icon file (e.g., `icon.svg`). |
| `tags`        | `string[]` | No       | Tags for filtering and search.                   |

**Example:**

```yaml
metadata:
  name: ollama-webui
  title: Ollama + Open WebUI
  description: Run large language models with a chat interface powered by Open WebUI.
  version: "1.0.0"
  category: ai-inference
  sourceRepo: https://github.com/swarm-cloud/marketplace-apps
  license: MIT
  icon: icon.svg
  tags:
    - ai
    - llm
    - inference
    - chat
```

---

### 2. Components

The `components` array is the core of a multi-chart application. Each entry defines a single deployable unit with its own source (Helm chart) and deployment configuration (values and patches). Parameters, UI, resources, and outputs remain global -- they are defined at the top level and shared across all components.

#### 2.1 Component Structure

Each component in the `components` array has the following fields:

| Field        | Type       | Required | Description                                                            |
|--------------|------------|----------|------------------------------------------------------------------------|
| `name`       | `string`   | Yes      | Unique name within the application. Must match `[a-z0-9-]+`.          |
| `source`     | `object`   | Yes      | Defines where the deployable artifact comes from. See [Source](#22-source). |
| `deployment` | `object`   | Yes      | Defines how parameters map to Helm values. See [Deployment](#23-deployment). |
| `dependsOn`  | `string[]` | No       | List of component names that must be deployed before this one.         |
| `images`     | `object[]` | No       | Allow-list of container images pinned by digest. See [Images](#27-images). |

**Example:**

```yaml
components:
  - name: ollama
    source: { ... }
    deployment: { ... }

  - name: open-webui
    dependsOn:
      - ollama
    source: { ... }
    deployment: { ... }
```

#### 2.2 Source

Defines where the deployable artifact comes from. Currently, only Helm charts are supported.

| Field     | Type     | Required | Description                              |
|-----------|----------|----------|------------------------------------------|
| `type`    | `string` | Yes      | Source type. Must be `helm`.             |
| `repoUrl` | `string` | Yes      | Helm chart repository URL.              |
| `chart`   | `string` | Yes      | Chart name within the repository.        |
| `version` | `string` | Yes      | Chart version (SemVer).                  |

> **Note:** Helm is used solely as a rendering engine. The platform invokes Helm to produce Kubernetes manifests but does not rely on Helm release management (Tiller, Helm secrets, etc.).

**Example:**

```yaml
components:
  - name: ollama
    source:
      type: helm
      repoUrl: https://otwld.github.io/ollama-helm/
      chart: ollama
      version: "1.12.0"
```

#### 2.3 Deployment

Each component's deployment section defines how user parameter values are transformed into Helm values for that specific chart.

**Structure:**

```yaml
deployment:
  values:
    base:
      # Static Helm values -- always applied
    patches:
      # Conditional modifications based on parameter values
```

**Base Values**

`base` is a plain YAML object that represents the default Helm values for the component. It is applied unconditionally before any patches.

```yaml
deployment:
  values:
    base:
      service:
        type: ClusterIP
      ollama:
        gpu:
          enabled: false
          number: 0
        models: []
      persistentVolume:
        enabled: true
        size: 30Gi
      ingress:
        enabled: false
```

**Patches**

Patches modify the base values conditionally based on parameter values. They are applied **in order** after the base values.

Each patch:

| Field   | Type     | Required | Description                                                  |
|---------|----------|----------|--------------------------------------------------------------|
| `op`    | `string` | Yes      | Operation: `add`, `replace`, or `remove`.                     |
| `path`  | `string` | Yes      | Dot-notation path to the target value.                        |
| `value` | `any`    | No       | Value to set. Required for `add` and `replace`. Supports expression interpolation. |
| `when`  | `string` | No       | Expression that must evaluate to `true` for the patch to apply. If omitted, the patch always applies. |

#### 2.4 Patch Operations

- **`add`** -- Sets the value at `path`. Creates intermediate objects if they don't exist.
- **`replace`** -- Replaces the value at `path`. The path must already exist in the resolved values.
- **`remove`** -- Removes the key at `path`.

**Example:**

```yaml
patches:
  - op: replace
    path: ollama.models
    value:
      - "{{ params.modelName }}"

  - op: replace
    path: ollama.gpu.enabled
    value: true
    when: "params.gpuEnabled == true"

  - op: replace
    path: ollama.gpu.number
    value: "{{ params.gpuCount }}"
    when: "params.gpuEnabled == true"

  - op: replace
    path: ingress.enabled
    value: true
    when: "params.enableIngress == true"

  - op: add
    path: ingress.hosts.0.host
    value: "{{ params.apiHostname }}"
    when: "params.enableIngress == true"
```

#### 2.5 Evaluation Order

For each component, values are evaluated as follows:

1. Start with a deep copy of `base`.
2. Iterate through `patches` in array order.
3. For each patch:
   a. If `when` is present, evaluate the expression. Skip if `false`.
   b. Resolve any `{{ }}` interpolation in `value`.
   c. Apply the operation.
4. The resulting object is passed to Helm as values for this component's chart.

#### 2.6 Dependencies & Deploy Order

Components are deployed in the order they appear in the `components` array. The `dependsOn` field declares explicit dependencies for cases where the platform needs to guarantee that one component is fully running before another is deployed.

**Rules:**

- If `dependsOn` is omitted, the component depends only on array ordering.
- `dependsOn` references MUST refer to component names earlier in the array.
- Circular dependencies are invalid and MUST be rejected at validation time.
- The platform MUST ensure that all components listed in `dependsOn` have completed deployment before starting the dependent component.

**Example:**

```yaml
components:
  - name: ollama
    source: { ... }
    deployment: { ... }

  - name: open-webui
    dependsOn:
      - ollama
    source: { ... }
    deployment: { ... }
```

In this example, `open-webui` will not begin deployment until `ollama` is fully deployed and running.

> **Note on node scheduling:** The platform handles node scheduling concerns (tolerations, nodeSelector, affinity) based on the `resources.gpu` declaration. Application definitions do not need to specify Kubernetes scheduling directives -- the platform injects them automatically when GPU resources are declared.

#### 2.7 Images

Optional allow-list of container images the component may run, pinned by digest:

```yaml
images:
  - name: ollama/ollama
    digest: sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
```

When declared:

- Every container image in the component's rendered manifests MUST appear in the list; rendering fails otherwise (fail closed).
- The platform rewrites floating tags to the pinned digests before deployment.
- Pinned digests make the rendered manifests fully deterministic, which makes the **expected evidence digest** computable from the definition alone -- shown in the listing's deployment blueprint before anything is deployed.

Declaring `images` is RECOMMENDED for every published application; marketplace review lanes may require it for verified listings.

#### 2.8 Deployment Evidence

After deployment, the platform computes a signed **deployment evidence** bundle from the rendered Kubernetes manifests: volatile fields are stripped by canonicalization rules, the surviving documents are digested (JCS / SHA-256) and signed. The AppDefinition does not describe the evidence itself -- it is derived entirely from what the components render.

Authors control what enters the snapshot through **annotations on the rendered resources**:

| Annotation | Effect |
|------------|--------|
| `swarm.io/exclude-from-evidence: "true"` | Drop the resource from the snapshot entirely. |
| `swarm.io/exclude-evidence-fields: "/p1,/p2"` | Strip the listed RFC 6901 JSON Pointer fields. |

These annotations are ordinary manifest content. They may originate from the chart's own templates, from `deployment.values.base`, or from patches -- the evidence layer does not care how they were produced, only what ends up in the rendered manifests. Example via base values (charts commonly expose `ingress.annotations`):

```yaml
deployment:
  values:
    base:
      ingress:
        annotations:
          swarm.io/exclude-from-evidence: "true"
```

**Transparency rules** (enforced by the evidence layer):

- Exclusions are **always visible as facts**: because the marketplace renders the manifests itself, it surfaces every exclusion on the listing and in the pre-deploy blueprint, and the `exclude-evidence-fields` annotation remains part of the evidence digest. It is impossible to hide *that* something is hidden.
- Helm values interpolating `sensitive` parameters or secret data-binding properties are stripped from evidence **automatically** and recorded as hidden fields -- authors do not (and cannot) opt out of this.
- Invalid pointers fail evidence computation (fail closed).

---

### 3. UI / Presentation

Controls how the deployment form is rendered. The UI section defines **visual structure only** -- it references parameters by `id` but does not define their schema or validation (that belongs in [Parameters](#4-parameters)).

#### 3.1 Sections

The form is organized into ordered sections.

| Field         | Type       | Required | Description                                  |
|---------------|------------|----------|----------------------------------------------|
| `id`          | `string`   | Yes      | Section identifier.                          |
| `title`       | `string`   | Yes      | Section heading.                             |
| `description` | `string`   | No       | Optional helper text below the heading.      |
| `fields`      | `Field[]`  | Yes      | Ordered list of fields in this section.      |

#### 3.2 Fields

Each field binds a parameter to a UI widget.

| Field        | Type     | Required | Description                                                 |
|--------------|----------|----------|-------------------------------------------------------------|
| `parameterId`| `string` | Yes      | References a parameter by `id`.                              |
| `widget`     | `string` | No       | Widget type override. Defaults based on parameter type.      |
| `visible`    | `string` | No       | Expression that controls visibility. See [Expressions](#7-expressions). |

#### 3.3 Widget Types

| Widget     | Description                     | Default for type         |
|------------|---------------------------------|--------------------------|
| `text`     | Single-line text input          | `string`                 |
| `textarea` | Multi-line text input           | --                       |
| `number`   | Numeric input with step         | `number`, `integer`      |
| `select`   | Dropdown selection              | any type with `options`  |
| `switch`   | Boolean toggle                  | `boolean`                |
| `slider`   | Range slider                    | `number` with min/max    |
| `password` | Masked text input               | --                       |

**Example:**

```yaml
ui:
  sections:
    - id: model
      title: Model Configuration
      description: Select the model to deploy.
      fields:
        - parameterId: modelName
          widget: select

    - id: compute
      title: Compute Resources
      fields:
        - parameterId: gpuEnabled
          widget: switch
        - parameterId: gpuCount
          widget: number
          visible: "params.gpuEnabled == true"

    - id: network
      title: Network Access
      fields:
        - parameterId: enableIngress
          widget: switch
        - parameterId: chatHostname
          widget: text
          visible: "params.enableIngress == true"
        - parameterId: apiHostname
          widget: text
          visible: "params.enableIngress == true"
```

---

### 4. Parameters

Parameters define the user-configurable inputs. They are global -- shared across all components. Each parameter has a schema, validation rules, and optional UI hints.

| Field          | Type       | Required | Description                                                      |
|----------------|------------|----------|------------------------------------------------------------------|
| `id`           | `string`   | Yes      | Unique identifier. Must match `[a-zA-Z][a-zA-Z0-9_]*`.          |
| `type`         | `string`   | Yes      | Data type: `string`, `number`, `integer`, `boolean`.             |
| `title`        | `string`   | Yes      | Display label.                                                   |
| `description`  | `string`   | No       | Help text shown below the field.                                 |
| `required`     | `boolean`  | No       | Whether the field must be filled. Default: `false`.              |
| `default`      | `any`      | No       | Default value. Must match `type`.                                |
| `sensitive`    | `boolean`  | No       | Marks the parameter as secret. Default: `false`. See [4.2](#42-secrets--lifecycle). |
| `generated`    | `boolean`  | No       | Generate a random value when left blank. Only for sensitive string parameters. Default: `false`. |
| `immutable`    | `boolean`  | No       | Value cannot be changed when reconfiguring an existing deployment. Default: `false`. |
| `options`      | `ValueSource` | No   | Defines available choices. See [Value Source](#5-options--value-source). |
| `validation`   | `object`   | No       | Validation constraints (see below).                              |

#### 4.1 Validation

Validation rules depend on the parameter type.

**String validation:**

| Field       | Type      | Description                          |
|-------------|-----------|--------------------------------------|
| `pattern`   | `string`  | Regex pattern the value must match.  |
| `minLength` | `integer` | Minimum string length.               |
| `maxLength` | `integer` | Maximum string length.               |

**Number / Integer validation:**

| Field       | Type     | Description                |
|-------------|----------|----------------------------|
| `min`       | `number` | Minimum value (inclusive).  |
| `max`       | `number` | Maximum value (inclusive).  |
| `step`      | `number` | Step increment for sliders. |

**Example:**

```yaml
parameters:
  - id: modelName
    type: string
    title: Model
    description: The LLM model to download and serve.
    required: true
    default: "llama3.1:8b"
    options:
      source:
        type: static
        values:
          - value: "llama3.2:3b"
            label: Llama 3.2 (3B)
          - value: "llama3.1:8b"
            label: Llama 3.1 (8B)
          - value: "mistral:7b"
            label: Mistral (7B)

  - id: gpuEnabled
    type: boolean
    title: Enable GPU
    description: Enable NVIDIA GPU acceleration for inference.
    default: true

  - id: gpuCount
    type: integer
    title: GPU Count
    description: Number of GPUs to allocate.
    default: 1
    validation:
      min: 1
      max: 8
```

#### 4.2 Secrets & Lifecycle

**Sensitive parameters** (`sensitive: true`) carry the same guarantees as secret data-binding properties:

- The value is sealed for the target TEE -- never visible to the host cloud, the marketplace, or the app author.
- The UI masks input (default widget: `password`).
- Helm values interpolating the parameter are stripped from the deployment evidence snapshot automatically and recorded as hidden fields -- see [Deployment Evidence](#28-deployment-evidence).
- The value may surface only through outputs of type `secret`.

**Generated values** (`generated: true`): if the user leaves the field blank, the platform generates a cryptographically random value. Only valid on sensitive string parameters.

**Immutable parameters** (`immutable: true`) cannot be changed when reconfiguring an existing deployment (e.g. persistent volume size). The reconfigure form renders them read-only.

```yaml
  - id: apiKey
    type: string
    title: API Key
    description: Bearer token clients must present. Auto-generated if left blank.
    sensitive: true
    generated: true

  - id: storageSize
    type: string
    title: Storage Size
    default: "30Gi"
    immutable: true
```

---

### 5. Options / Value Source

The `ValueSource` abstraction defines where a parameter's available options come from.

#### 5.1 Structure

```yaml
options:
  source:
    type: static | remote    # "remote" reserved for future use
    values: [...]            # required when type is "static"
```

#### 5.2 Static Source

A static source embeds the option list directly in the app definition.

| Field    | Type          | Required | Description                         |
|----------|---------------|----------|-------------------------------------|
| `type`   | `string`      | Yes      | Must be `static`.                   |
| `values` | `Option[]`    | Yes      | List of available options.          |

Each **Option**:

| Field   | Type     | Required | Description                      |
|---------|----------|----------|----------------------------------|
| `value` | `any`    | Yes      | The value stored when selected.  |
| `label` | `string` | Yes      | Display text in the UI.          |

#### 5.3 Remote Source (Future)

> **Reserved.** The `remote` source type is defined for forward compatibility. It will allow fetching options from an API endpoint at form render time. The exact schema will be defined in a future spec version.

```yaml
# Future example -- NOT yet supported
options:
  source:
    type: remote
    url: "https://api.example.com/models"
    valuePath: "$.models[*].id"
    labelPath: "$.models[*].name"
    cacheTtl: 300
```

---

### 6. Resource Requirements

Declares the compute resources an application needs. The platform uses this information to:

- **Schedule** the application onto cluster spaces with sufficient capacity.
- **Display** resource requirements in the marketplace UI before deployment.
- **Validate** that the target cluster can accommodate the workload.

Resource requirements are global -- they describe the total resources needed by all components combined.

#### 6.1 Structure

```yaml
resources:
  min:
    cpu: "4"
    memory: "8Gi"
    storage: "30Gi"
  recommended:
    cpu: "8"
    memory: "16Gi"
    storage: "50Gi"
  gpu:
    required: false
    minCount: 1
    minVram: "8Gi"
    types:
      - nvidia-a100
      - nvidia-l40s
      - nvidia-rtx-4090
```

#### 6.2 Fields

**`resources.min`** -- Minimum resources required to run the application. Deployment MUST be rejected if the target cluster cannot satisfy these.

| Field     | Type     | Required | Description                                          |
|-----------|----------|----------|------------------------------------------------------|
| `cpu`     | `string` | Yes      | Minimum CPU cores (Kubernetes quantity format).       |
| `memory`  | `string` | Yes      | Minimum memory (e.g., `4Gi`, `512Mi`).               |
| `storage` | `string` | No       | Minimum persistent storage.                          |

**`resources.recommended`** -- Recommended resources for optimal performance. Shown to the user as guidance.

| Field     | Type     | Required | Description                                          |
|-----------|----------|----------|------------------------------------------------------|
| `cpu`     | `string` | No       | Recommended CPU cores.                               |
| `memory`  | `string` | No       | Recommended memory.                                  |
| `storage` | `string` | No       | Recommended persistent storage.                      |

**`resources.gpu`** -- GPU requirements. When declared, the platform automatically handles Kubernetes node scheduling (tolerations, nodeSelector, affinity) to place workloads on GPU-capable nodes. Application definitions do not need to specify scheduling directives.

| Field      | Type       | Required | Description                                         |
|------------|------------|----------|-----------------------------------------------------|
| `required` | `boolean`  | No       | Whether a GPU is mandatory. Default: `false`.        |
| `minCount` | `integer`  | No       | Minimum number of GPUs.                              |
| `minVram`  | `string`   | No       | Minimum VRAM per GPU (e.g., `8Gi`).                 |
| `types`    | `string[]` | No       | List of acceptable GPU types.                        |

> **Note on dynamic resources:** Some parameters (e.g., `gpuEnabled`, `gpuCount`) may affect actual resource consumption at runtime. The `resources` section defines the **baseline** for scheduling. Runtime resource allocation is handled by the deployment layer, not this specification.

---

### 7. Expressions

Expressions are a lightweight language used in `when` conditions, `visible` rules, and value interpolation. They are intentionally minimal -- this is **not** a general-purpose programming language.

#### 7.1 Namespaces

The expression system supports the following namespaces:

| Namespace                       | Description                                              |
|---------------------------------|----------------------------------------------------------|
| `params.<id>`                   | User-provided parameter value.                           |
| `components.<name>.release`     | Helm release name assigned by the platform for the named component. |
| `namespace`                     | Kubernetes namespace where the application is deployed (platform-provided). |
| `data.<slot>`                   | Resolved data binding for the named slot (count, dataset, interface, connection properties). See [Binding Resolution](#binding-resolution). |
| `secrets.<name>`                | Publisher secret declared in the definition's `secrets` list; the value lives in the owner's console vault. See [Publisher Secrets](#publisher-secrets). |
| `binding.<field>`               | Binding context for credential provisioners (`action`, `deploymentId`, `organization`, `app`, `evidenceDigest`). Provisioner environments only. See [Binding Context](#binding-context). |

#### 7.2 Syntax

**Variable access:**

```
params.<parameterId>
components.<componentName>.release
namespace
data.<slotName>.count
data.<slotName>.dataset
data.<slotName>.interface
data.<slotName>.connection.<property>
secrets.<name>
binding.<field>
```

**Comparison operators:**

| Operator | Description        | Example                        |
|----------|--------------------|--------------------------------|
| `==`     | Equal              | `params.gpuEnabled == true`    |
| `!=`     | Not equal          | `params.modelName != ""`       |
| `>`      | Greater than       | `params.gpuCount > 1`          |
| `>=`     | Greater or equal   | `params.gpuCount >= 2`         |
| `<`      | Less than          | `params.gpuCount < 4`          |
| `<=`     | Less or equal      | `params.gpuCount <= 8`         |

**Logical operators:**

| Operator | Description | Example                                          |
|----------|-------------|--------------------------------------------------|
| `&&`     | Logical AND | `params.gpuEnabled == true && params.gpuCount > 1` |
| `\|\|`   | Logical OR  | `params.enableIngress == true \|\| params.chatHostname != ""` |

**String interpolation (in `value` fields):**

```
"{{ params.modelName }}"
"http://{{ components.ollama.release }}:11434"
"http://{{ components.ollama.release }}.{{ namespace }}.svc.cluster.local:11434"
"clickhouse://{{ data.feedback.connection.host }}:{{ data.feedback.connection.port }}"
"{{ data.corpus | json }}"
```

Double curly braces `{{ }}` interpolate a value into a string. This supports all namespaces from [7.1](#71-namespaces).

**Filters:** the expression language supports exactly one filter, `| json`, which serializes an entire slot binding to JSON. See [The `json` Filter](#the-json-filter).

#### 7.3 Scope

Expressions can be used in the following contexts:

| Context                          | Field                                     | Allowed namespaces                                    | Purpose                          |
|----------------------------------|-------------------------------------------|-------------------------------------------------------|----------------------------------|
| Base values                      | `deployment.values.base`                  | `params`, `components`, `namespace`, `data`, `secrets` | Inter-component references.      |
| Patch value interpolation        | `deployment.values.patches[].value`       | `params`, `components`, `namespace`, `data`, `secrets` | Inject parameter and binding values. |
| Patch conditions                 | `deployment.values.patches[].when`        | `params`, `data` (only `count` / `interface`)         | Conditionally apply patches.     |
| UI field visibility              | `ui.sections[].fields[].visible`          | `params`                                              | Show/hide form fields.           |
| Output value templates           | `outputs[].value`                         | `params`, `components`, `namespace`, `data` (non-secret only) | Compute output values.    |
| Output conditions                | `outputs[].when`                          | `params`, `data` (only `count` / `interface`)         | Conditionally show outputs.      |
| Static connection templates      | `spec.provides[].connection.static`       | `secrets`                                             | Fill connection properties at publication render. |
| Provisioner environment          | `spec.provides[].connection.provisioner.env` | `secrets`, `binding`                               | Inputs for per-binding credential minting. |

> **Note:** Conditions (`when`, `visible`) only support values known at configure time. Component release names, namespace, and connection properties are resolved at deploy time, so they are excluded from conditions. Of the `data` namespace, only `data.<slot>.count` and `data.<slot>.interface` are known at configure time (the user has selected datasets, but credentials are not yet brokered) and are therefore allowed in `when`. `visible` remains `params`-only because the Data step is rendered after the parameter sections. Secret connection properties and `secrets.*` values MUST NOT be used in outputs or conditions -- see [Secrets & Attestation](#secrets--attestation). The `binding` namespace exists only inside provisioner environments.

#### 7.4 Limitations

- No function calls.
- No filters other than `| json`.
- No arithmetic operations.
- No nested object access beyond the supported namespaces.
- No ternary or conditional expressions.
- Boolean values in comparisons must use `true` / `false` (not `1` / `0`).

---

### 8. Outputs

Outputs define values that the platform exposes to the user **after** a successful deployment. They describe how to derive user-facing information from the running application.

#### 8.1 Structure

```yaml
outputs:
  - id: chatUrl
    title: Chat UI
    type: url
    value: "https://{{ params.chatHostname }}"
    when: "params.enableIngress == true"

  - id: internalApi
    title: Internal API
    type: url
    value: "http://{{ components.ollama.release }}.{{ namespace }}.svc.cluster.local:11434"
```

#### 8.2 Fields

| Field   | Type     | Required | Description                                             |
|---------|----------|----------|---------------------------------------------------------|
| `id`    | `string` | Yes      | Unique output identifier.                                |
| `title` | `string` | Yes      | Display label.                                           |
| `type`  | `string` | No       | Output type hint: `url`, `string`, `secret`. Default: `string`. |
| `value` | `string` | Yes      | Value template. Supports `{{ params.<id> }}`, `{{ components.<name>.release }}`, and `{{ namespace }}`. |
| `when`  | `string` | No       | Expression controlling whether this output is shown. Only `params` namespace is allowed. |

---

### 9. Data Slots

Applications that consume marketplace datasets declare **data slots** in the optional top-level `data` section. A slot is a named socket that the user plugs datasets into at configure time -- analogous to a volume mount. The slot declares *how* the application can connect (accepted [data interfaces](#data-interface-specification) -- the protocol layer) and *what shape* of data it expects (schema constraints -- the semantic layer). A slot never references concrete datasets.

#### 9.1 Structure

```yaml
data:
  slots:
    - name: corpus
      title: Knowledge Base
      description: Datasets the agent retrieves answers from.
      required: true
      multiple: true
      accepts:
        - interface: clickhouse-native/v1
        - interface: s3-parquet/v1
      schema:
        tables:
          - columns:
              - name: text
                type: string
```

#### 9.2 Slot Fields

| Field         | Type       | Required | Description                                                                 |
|---------------|------------|----------|-----------------------------------------------------------------------------|
| `name`        | `string`   | Yes      | Slot identifier. Must match `[a-zA-Z][a-zA-Z0-9_]*`. Used in the `data.<slot>` expression namespace. |
| `title`       | `string`   | Yes      | Display label in the configure flow.                                        |
| `description` | `string`   | No       | Help text shown below the dataset picker.                                   |
| `required`    | `boolean`  | No       | Whether at least one dataset must be bound before deployment. Default: `false`. |
| `multiple`    | `boolean`  | No       | Whether more than one dataset may be bound. Default: `false`.               |
| `accepts`     | `object[]` | Yes      | Ordered list of accepted interfaces (`{ interface: <name>/<version> }`). Order defines negotiation preference -- see [Compatibility Matching](#compatibility-matching). |
| `schema`      | `object`   | No       | Semantic constraints a bound dataset must satisfy.                          |

#### 9.3 Schema Constraints

`schema.tables` is a list of table constraints:

| Field     | Type               | Required | Description                                                                    |
|-----------|--------------------|----------|--------------------------------------------------------------------------------|
| `name`    | `string`           | No       | If set, the constraint must be satisfied by the dataset table with this exact name. If omitted, any table qualifies. |
| `columns` | `RequiredColumn[]` | Yes      | Columns that must exist. Each entry is `{ name, type }` plus optional `items` / `dimensions`, using the canonical [column types](#column-types). |

Matching semantics are defined in [Compatibility & Binding](#compatibility--binding).

#### 9.4 UI

Slots are not referenced from `ui.sections`. The platform automatically renders a dedicated **Data** step in the configure flow after the parameter sections, listing slots in declaration order. Each slot is rendered as a dataset picker filtered to datasets that are both [compatible](#compatibility-matching) and accessible to the deploying organization.

#### 9.5 Consuming Bindings

Bound datasets are exposed to `deployment.values` through the `data.<slot>` expression namespace, resolved at deploy time:

```yaml
patches:
  - op: add
    path: datasources.feedback.url
    value: "clickhouse://{{ data.feedback.connection.host }}:{{ data.feedback.connection.port }}/{{ data.feedback.connection.database }}"
    when: "data.feedback.count > 0"

  - op: replace
    path: datasources.corpusJson
    value: "{{ data.corpus | json }}"
```

See [Binding Resolution](#binding-resolution) for the full namespace shape.

---

## Data Interface Specification

A **DataInterface** is a named, versioned contract that describes one way of connecting to data -- the **protocol layer** of data compatibility. Interfaces decouple the two sides of the market: applications declare which interfaces they *accept* (per slot), datasets declare which interfaces they *provide*, and neither side ever references the other directly.

Interfaces live in the marketplace registry under `interfaces/<name>/<version>.yaml`, one file per contract version.

### Structure

```yaml
apiVersion: swarm.cloud/v1alpha1
kind: DataInterface

metadata:
  name: clickhouse-native
  version: v1
  title: ClickHouse Native Protocol
  description: Direct access to a ClickHouse database over the native TCP protocol.

spec:
  connection:
    properties:
      host:
        type: string
        description: ClickHouse server hostname.
      port:
        type: integer
        default: 9440
        description: Native protocol port (TLS).
      database:
        type: string
        description: Database containing the dataset tables.
      username:
        type: string
        description: Read-only user provisioned for this binding.
      password:
        type: string
        secret: true
        description: Password for the binding user. Sealed for the target TEE.
      tls:
        type: boolean
        default: true
        description: Whether to connect over TLS.
    required:
      - host
      - port
      - database
      - username
      - password
```

### Interface Metadata

| Field         | Type     | Required | Description                                                              |
|---------------|----------|----------|--------------------------------------------------------------------------|
| `name`        | `string` | Yes      | Unique interface identifier. Must match `[a-z0-9-]+`.                    |
| `version`     | `string` | Yes      | Contract version (`v1alpha1`, `v1beta1`, `v1`, ...), independent of `apiVersion`. |
| `title`       | `string` | Yes      | Human-readable display name.                                             |
| `description` | `string` | Yes      | Short description of the connection method.                              |

### Connection Properties

`spec.connection.properties` is a map from property name to definition. It defines the exact shape of the resolved [binding object](#binding-resolution) that applications consume through `data.<slot>.connection.<property>`.

| Field         | Type      | Required | Description                                                            |
|---------------|-----------|----------|------------------------------------------------------------------------|
| `type`        | `string`  | Yes      | `string`, `integer`, `number`, or `boolean`.                            |
| `description` | `string`  | No       | Human-readable description.                                             |
| `default`     | `any`     | No       | Default value. Must match `type`.                                       |
| `secret`      | `boolean` | No       | Marks the property as secret. Default: `false`. See [Secrets & Attestation](#secrets--attestation). |

`spec.connection.required` lists property names that MUST be present in every resolved binding.

### Referencing an Interface

App slots (`accepts`) and datasets (`provides`) reference interfaces as `<name>/<version>`:

```yaml
accepts:
  - interface: clickhouse-native/v1
```

### Evolution Rules

- Adding an **optional** connection property is allowed within a version.
- Adding a required property, removing a property, or changing a property's type is a **breaking change** and requires publishing a new version file. Versions coexist in the registry; apps and datasets migrate independently.

### Initial Registry

| Interface              | Description                                                        |
|------------------------|--------------------------------------------------------------------|
| `clickhouse-native/v1` | Direct access to a ClickHouse database over the native TCP protocol. |
| `s3-parquet/v1`        | Read access to Parquet files in an S3-compatible bucket.           |
| `s3-jsonl/v1`          | Read access to JSON Lines files in an S3-compatible bucket.        |
| `s3-csv/v1`            | Read access to CSV files in an S3-compatible bucket.               |
| `s3-npy/v1`            | Read access to NumPy array files (embeddings) in an S3-compatible bucket. |

---

## Data Definition Specification

A **DataDefinition** describes a dataset published to the marketplace: what it contains (semantic layer), how it can be reached (provided interfaces), and how connection objects are produced ([connection resolution](#connection-resolution)). It contains **no secret values and no access rules** -- secrets are referenced by name and stored in the owner's console vault ([Publisher Secrets](#publisher-secrets)); visibility and usage terms are configured on the marketplace listing.

Datasets are typically published from the organization admin console (from ClickHouse or S3-compatible sources); the platform materializes a DataDefinition for each. The `datasets/` directory hosts curated public definitions. The contract is identical either way.

### Structure

```yaml
apiVersion: swarm.cloud/v1alpha1
kind: DataDefinition

metadata:
  name: retail-transactions
  title: Retail Transactions 2024
  description: Anonymized point-of-sale transactions from a European retail chain, updated daily.
  category: retail
  tags:
    - retail
    - transactions
    - timeseries

spec:
  secrets:
    - name: chReadUser
      title: ClickHouse read-only user
    - name: chReadPass
      title: ClickHouse read-only password
    - name: awsAdminKeyId
      title: AWS access key ID (IAM admin)
    - name: awsAdminSecret
      title: AWS secret access key (IAM admin)

  provides:
    - interface: clickhouse-native/v1
      connection:
        static:
          host: ch.acme-data.internal
          port: 9440
          database: retail
          username: "{{ secrets.chReadUser }}"
          password: "{{ secrets.chReadPass }}"
          tls: true

    - interface: s3-parquet/v1
      connection:
        provisioner:
          image: acme/s3-creds-minter@sha256:b5bb9d8014a0f9b1d61e21e796d78dccdf1352f23cd32812f4850b878ae4944c
          env:
            ACTION: "{{ binding.action }}"
            DEPLOYMENT_ID: "{{ binding.deploymentId }}"
            CONSUMER_ORG: "{{ binding.organization }}"
            CONSUMER_APP: "{{ binding.app }}"
            EVIDENCE_DIGEST: "{{ binding.evidenceDigest }}"
            AWS_ACCESS_KEY_ID: "{{ secrets.awsAdminKeyId }}"
            AWS_SECRET_ACCESS_KEY: "{{ secrets.awsAdminSecret }}"

  schema:
    tables:
      - name: transactions
        description: One row per purchased line item.
        columns:
          - name: ts
            type: timestamp
            description: Purchase time (UTC).
          - name: sku
            type: string
            description: Stock keeping unit.
          - name: amount
            type: number
            description: Line total in EUR.
          - name: text
            type: string
            nullable: true
            description: Free-text product description.
```

### Dataset Metadata

Same shape as application metadata: `name`, `title`, `description` (required); `category`, `icon`, `tags` (optional). Datasets do not carry `version`, `sourceRepo`, or `license` -- freshness and size are runtime statistics, and usage terms are listing policy.

### Provides

`spec.provides` lists the interfaces the platform can serve this dataset over. The platform MUST be able to fulfill every declared interface (e.g., a ClickHouse-backed dataset additionally exported as Parquet snapshots to satisfy `s3-parquet/v1`). Order is not significant -- negotiation preference is defined by the app slot's `accepts` order. Each entry SHOULD declare a [connection resolution](#connection-resolution).

### Connection Resolution

Each entry in `provides` declares how its connection object is produced. The marketplace treats data interfaces as opaque, community-defined types: the only rule the platform enforces is that **whatever the resolution produces MUST validate against the connection schema of the negotiated interface**. This keeps the platform agnostic -- the community can add any number of interfaces and provisioners without platform changes.

Two mechanisms:

**Static** -- connection properties rendered once at publication, from literals and [publisher secrets](#publisher-secrets). Every binding receives the same values (shared credentials):

```yaml
provides:
  - interface: clickhouse-native/v1
    connection:
      static:
        host: ch.acme-data.internal
        port: 9440
        database: retail
        username: "{{ secrets.chReadUser }}"
        password: "{{ secrets.chReadPass }}"
        tls: true
```

**Provisioner** -- a container the platform executes for every binding, minting individual credentials per consumer:

```yaml
provides:
  - interface: s3-parquet/v1
    connection:
      provisioner:
        image: acme/s3-creds-minter@sha256:b5bb9d8014a0f9b1d61e21e796d78dccdf1352f23cd32812f4850b878ae4944c
        env:
          ACTION: "{{ binding.action }}"
          DEPLOYMENT_ID: "{{ binding.deploymentId }}"
          EVIDENCE_DIGEST: "{{ binding.evidenceDigest }}"
          AWS_ACCESS_KEY_ID: "{{ secrets.awsAdminKeyId }}"
          AWS_SECRET_ACCESS_KEY: "{{ secrets.awsAdminSecret }}"
```

Static credentials mean revoking one consumer requires rotating everyone; provisioner-minted credentials are individually issued, revocable, and auditable. The marketplace surfaces which mode a dataset uses as a listing fact.

#### Provisioner Execution Contract

A provisioner run is a pure function `env → connection JSON`, executed in an isolated sandbox on marketplace infrastructure. The data owner needs no infrastructure of their own -- their data can live in any external service (a plain S3 bucket, a managed ClickHouse), and the provisioner is just a container that talks to that service's API.

- `image` MUST be digest-pinned. The platform releases sealed secret values only into the attested sandbox running exactly that image.
- Inputs arrive exclusively through environment variables; values may interpolate `secrets.<name>` and `binding.<field>`.
- The container MUST write the connection object as JSON to `/swarm/output/connection.json` and exit `0`. The platform validates the object against the negotiated interface's connection schema, seals it for the consumer's TEE, and exposes it as `data.<slot>.connection.*`.
- The sandbox provides no Kubernetes API, no platform credentials, no inbound connectivity, and no access to other workloads; runs are resource-capped and time-boxed. Non-zero exit, timeout, or schema-invalid output fails the binding (fail closed).
- The same image is invoked with `binding.action = provision` when a binding is created and `binding.action = revoke` when it is removed (undeploy, slot unbind, or access revocation). Both operations MUST be idempotent -- the platform may retry.
- Every run is recorded in the owner's activity log together with the image digest and binding context.

#### Binding Context

The `binding` namespace is available only inside `provisioner.env`:

| Field                    | Description                                                              |
|--------------------------|--------------------------------------------------------------------------|
| `binding.action`         | `provision` or `revoke`.                                                 |
| `binding.deploymentId`   | Unique identifier of the consuming deployment.                           |
| `binding.organization`   | Name of the consuming organization.                                      |
| `binding.app`            | Name of the consuming application.                                       |
| `binding.evidenceDigest` | Evidence digest of the consuming deployment. Lets the provisioner verify the consumer against the owner's expectations before minting. |

Minted credentials SHOULD embed the binding identity (e.g. an IAM user named after `binding.deploymentId`) so the owner's access logs attribute every read to a specific deployment.

### Schema (Semantic Layer)

`spec.schema` declares the dataset's tables and typed columns. This is the machine-readable counterpart of the human-facing "Columns" view in the marketplace UI, and the input to [schema matching](#compatibility-matching).

Each table: `name` (required), `description`, `columns` (required, min 1). Each column: `name`, `type` (required), `description`, `nullable` (default `false`), plus `items` (element type, for `array` columns) and `dimensions` (vector length, for `vector` columns).

#### Column Types

| Type        | Description                                    |
|-------------|------------------------------------------------|
| `string`    | Text of any length.                            |
| `integer`   | Whole number.                                  |
| `number`    | Decimal / floating-point number.               |
| `boolean`   | True / false.                                  |
| `timestamp` | Point in time (date + time, UTC unless noted). |
| `date`      | Calendar date without time.                    |
| `json`      | Structured JSON value.                         |
| `binary`    | Opaque binary payload.                         |
| `array`     | Ordered list of values of a single element type, declared via `items`. |
| `vector`    | Fixed-length numeric embedding, length declared via `dimensions`. |

Types are canonical: the publisher maps the storage engine's native types (e.g., ClickHouse `DateTime64`, Parquet `INT64`, NumPy `float32[768]`) onto this set.

Datasets SHOULD declare a schema. A dataset without one cannot match any schema-constrained slot.

### Access & Usage Terms

Access control is **not part of the definition**. Visibility (public vs. restricted), usage terms, per-organization and per-application grants, and the request-and-approval flow are configured by the dataset owner on the marketplace listing and enforced by the platform at configure and deploy time. Keeping them out of the source-stored definition means access can change instantly -- granting or revoking a consumer never requires republishing the dataset. See [Definitions Describe Facts, Listings Describe Policy](#definitions-describe-facts-listings-describe-policy).

---

## Publisher Secrets

Applications and datasets may declare **publisher secrets** -- values the publishing organization enters in its admin console, referenced from the definition by name via `{{ secrets.<name> }}`. The definition carries only the declaration; the values live in the organization's vault and never appear in Git, listings, or evidence plaintext.

Declared at the top level in `app.yaml` and under `spec` in `data.yaml`:

```yaml
secrets:
  - name: licenseKey
    title: License key
    description: Issued with your commercial subscription.
```

| Field         | Type     | Required | Description                                        |
|---------------|----------|----------|----------------------------------------------------|
| `name`        | `string` | Yes      | Identifier used as `secrets.<name>` in expressions. Must match `[a-zA-Z][a-zA-Z0-9_]*`. |
| `title`       | `string` | Yes      | Label shown in the publication form.               |
| `description` | `string` | No       | Help text shown in the publication form.           |

Rules:

- On publication, the console prompts the owner for exactly the declared secrets -- the form is generated from the declaration, the same way the deployment form is generated from `parameters`.
- Referencing an undeclared secret, or publishing with a declared secret left unset, fails the publication (fail closed).
- Secret values carry the same guarantees as `sensitive` parameters: sealed for the TEE that consumes them, never visible to deploying users or the marketplace operator, automatically stripped from deployment evidence and recorded as hidden fields, forbidden in outputs and conditions.
- Typical uses: license keys or private-registry tokens in application deployment values; storage admin credentials in dataset [connection resolution](#connection-resolution).

---

## Compatibility & Binding

This chapter defines how the platform decides which datasets fit which applications, and how a selected dataset becomes concrete configuration inside a deployment.

### Compatibility Matching

A dataset is **compatible** with a slot when both checks pass:

**1. Protocol match.** The *negotiated interface* is the first entry in the slot's `accepts` list that also appears in the dataset's `provides`. If no common interface exists, the pair is incompatible.

**2. Schema match.** Applies only when the slot declares `schema` constraints:

- A dataset with no declared schema fails every schema-constrained slot.
- Every table constraint must be satisfied by a **distinct** dataset table.
- A constraint with `name` must be satisfied by the table of that exact name; without `name`, any table qualifies.
- A table satisfies a constraint when, for every required column, it has a column with the same name and a compatible type.

Type compatibility:

| Required type | Satisfied by declared type |
|---------------|----------------------------|
| `number`      | `number`, `integer`        |
| `array`       | `array`; if the constraint sets `items`, element types must match exactly |
| `vector`      | `vector`; if the constraint sets `dimensions`, lengths must match exactly |
| any other     | exact match                |

Both checks depend only on declarations, so compatibility is computed **statically** -- the marketplace shows "compatible datasets" on an app listing (and vice versa) before anything is deployed.

**Compatibility vs. access.** These are orthogonal axes: matching says a pair *can* work together; access grants -- configured on the marketplace listing, outside this spec -- say it *may*. Configure-time pickers show `compatible ∩ granted`; compatible-but-ungranted datasets surface with a request-access affordance.

### Binding Resolution

At configure time the user fills each slot with datasets. At deploy time the platform resolves each filled slot into a **binding**: connection properties come from the dataset's [connection resolution](#connection-resolution) -- static connections are taken from the publication render, provisioner-backed connections are minted per binding. The resolved binding is exposed through the `data` expression namespace:

| Expression                          | Available for | Description                                                        |
|-------------------------------------|---------------|--------------------------------------------------------------------|
| `data.<slot>.count`                 | all slots     | Number of bound datasets. `0` or `1` for single slots.             |
| `data.<slot>.dataset`               | single slots  | Name of the bound dataset (`metadata.name`).                        |
| `data.<slot>.interface`             | single slots  | Negotiated interface reference, e.g. `clickhouse-native/v1`.        |
| `data.<slot>.connection.<property>` | single slots  | Resolved connection property, as defined by the negotiated interface. |
| `data.<slot> \| json`               | all slots     | Entire binding serialized as JSON.                                  |

For `multiple: true` slots, per-property access is not available -- inject the whole slot via the `json` filter. Accessing properties of an empty optional slot is a deploy-time error; guard such patches with `when: "data.<slot>.count > 0"`.

### The `json` Filter

`{{ data.<slot> | json }}` serializes the resolved binding. It is the only filter in the expression language.

- Single slot (`multiple: false`): a single object, or `null` when the slot is empty.
- Multi slot (`multiple: true`): an array (possibly empty), one object per bound dataset.

Each object has the shape:

```json
{
  "dataset": "retail-transactions",
  "interface": "clickhouse-native/v1",
  "connection": {
    "host": "…",
    "port": 9440,
    "database": "…",
    "username": "…",
    "password": "…",
    "tls": true
  }
}
```

A value produced by `| json` that includes secret connection properties is treated as secret in its entirety -- see below.

### Secrets & Attestation

Data bindings are where the marketplace meets the confidential-computing layer:

- Any Helm value that interpolates a `secret: true` connection property (directly or via `| json`) is **sealed for the target TEE**. It never appears in plaintext in the deployment blueprint or evidence bundle -- evidence carries only a digest.
- The platform brokers credentials: the dataset owner's credentials are released to a workload **only after** the deployment's evidence has been verified (the owner's gatekeeper policy). Neither the app author, nor the deploying user, nor the marketplace ever sees them.
- At deploy time the platform MUST verify that the dataset owner has granted access to the deploying organization and to the specific application being deployed. Compatibility alone is not authorization.
- Secret connection properties MUST NOT be referenced in `outputs` and cannot appear in conditions.

### Future Extensions

Reserved for future spec versions, following the same forward-compatibility rules as the rest of this document:

- **Access modes** -- slots declaring `read` vs `readwrite` intent, matched against modes the owner allows on the listing.
- **Semantic profiles** -- named, registry-managed schema profiles (e.g. `text-corpus/v1`) as a shorthand for structural constraints.
- **Credential TTL & rotation** -- short-lived credentials re-minted by long-running issuers instead of one-shot provisioner runs.

---

## Versioning

### Spec Version

Every document -- `AppDefinition`, `DataDefinition`, and `DataInterface` -- declares its spec version via the `apiVersion` field:

```yaml
apiVersion: swarm.cloud/v1alpha1
```

The format is `swarm.cloud/<version>` where `<version>` follows Kubernetes API versioning conventions:

| Stage       | Meaning                                              |
|-------------|------------------------------------------------------|
| `v1alpha1`  | Experimental. May have breaking changes.             |
| `v1beta1`   | Stabilizing. Breaking changes are discouraged.       |
| `v1`        | Stable. Breaking changes require a new API version.  |

### Compatibility

- The platform MUST validate `apiVersion` before processing.
- Unknown `apiVersion` values MUST be rejected.
- Fields not recognized by the current spec version SHOULD be ignored (forward compatibility).
- Removing a required field in a new version is a breaking change and requires a version bump.

### App Version

The user-facing application version is declared in `metadata.version` (SemVer) and is what the marketplace shows in the listing's version history. Component chart versions (`source.version`) and pinned image digests remain the deployable artifact versions; changing any of them SHOULD bump `metadata.version`.

### Interface Version

Data interface contracts carry their own version in `metadata.version` (`v1`, `v2beta1`, ...), independent of the spec's `apiVersion`, and follow the same staging conventions. Evolution rules are defined in [Data Interface Specification](#data-interface-specification): optional additions are allowed within a version; any breaking change requires a new version file, and versions coexist in the registry.

---

## Full Examples

Live examples in this repository: [`apps/ollama-webui/app.yaml`](apps/ollama-webui/app.yaml), [`interfaces/clickhouse-native/v1.yaml`](interfaces/clickhouse-native/v1.yaml), [`interfaces/s3-parquet/v1.yaml`](interfaces/s3-parquet/v1.yaml), [`datasets/retail-transactions/data.yaml`](datasets/retail-transactions/data.yaml).

### Application: Ollama + Open WebUI

A complete `app.yaml` for deploying Ollama with Open WebUI -- a multi-chart application with two components.

```yaml
apiVersion: swarm.cloud/v1alpha1
kind: AppDefinition

metadata:
  name: ollama-webui
  title: Ollama + Open WebUI
  description: Run large language models with a chat interface powered by Open WebUI.
  version: "1.0.0"
  category: ai-inference
  sourceRepo: https://github.com/swarm-cloud/marketplace-apps
  license: MIT
  icon: icon.svg
  tags:
    - ai
    - llm
    - inference
    - chat

ui:
  sections:
    - id: model
      title: Model Configuration
      description: Select the model to deploy.
      fields:
        - parameterId: modelName
          widget: select
    - id: compute
      title: Compute Resources
      fields:
        - parameterId: gpuEnabled
          widget: switch
        - parameterId: gpuCount
          widget: number
          visible: "params.gpuEnabled == true"
    - id: storage
      title: Storage
      fields:
        - parameterId: storageSize
          widget: select
    - id: network
      title: Network Access
      fields:
        - parameterId: enableIngress
          widget: switch
        - parameterId: chatHostname
          widget: text
          visible: "params.enableIngress == true"
        - parameterId: apiHostname
          widget: text
          visible: "params.enableIngress == true"

parameters:
  - id: modelName
    type: string
    title: Model
    description: The LLM model to download and serve.
    required: true
    default: "llama3.1:8b"
    options:
      source:
        type: static
        values:
          - value: "llama3.2:3b"
            label: Llama 3.2 (3B)
          - value: "llama3.1:8b"
            label: Llama 3.1 (8B)
          - value: "mistral:7b"
            label: Mistral (7B)
          - value: "gemma2:9b"
            label: Gemma 2 (9B)
          - value: "qwen2.5:7b"
            label: Qwen 2.5 (7B)
          - value: "deepseek-r1:14b"
            label: DeepSeek R1 (14B)

  - id: gpuEnabled
    type: boolean
    title: Enable GPU
    description: Enable NVIDIA GPU acceleration for inference.
    default: true

  - id: gpuCount
    type: integer
    title: GPU Count
    description: Number of GPUs to allocate.
    default: 1
    validation:
      min: 1
      max: 8

  - id: storageSize
    type: string
    title: Storage Size
    description: Persistent storage for downloaded models.
    default: "30Gi"
    options:
      source:
        type: static
        values:
          - value: "20Gi"
            label: 20 GB
          - value: "30Gi"
            label: 30 GB
          - value: "50Gi"
            label: 50 GB
          - value: "100Gi"
            label: 100 GB

  - id: enableIngress
    type: boolean
    title: Enable Public Access
    description: Expose services via ingress endpoints.
    default: false

  - id: chatHostname
    type: string
    title: Chat UI Hostname
    description: Public hostname for the Open WebUI chat interface.
    validation:
      pattern: "^[a-z0-9]([a-z0-9-]*[a-z0-9])?(\\.[a-z0-9]([a-z0-9-]*[a-z0-9])?)*$"

  - id: apiHostname
    type: string
    title: API Hostname
    description: Public hostname for the Ollama API (OpenAI-compatible).
    validation:
      pattern: "^[a-z0-9]([a-z0-9-]*[a-z0-9])?(\\.[a-z0-9]([a-z0-9-]*[a-z0-9])?)*$"

resources:
  min:
    cpu: "4"
    memory: "8Gi"
    storage: "30Gi"
  recommended:
    cpu: "8"
    memory: "16Gi"
    storage: "50Gi"
  gpu:
    required: false
    minCount: 1
    minVram: "8Gi"
    types:
      - nvidia-a100
      - nvidia-l40s
      - nvidia-rtx-4090

components:
  - name: ollama
    source:
      type: helm
      repoUrl: https://otwld.github.io/ollama-helm/
      chart: ollama
      version: "1.12.0"
    images:
      - name: ollama/ollama
        digest: sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
    deployment:
      values:
        base:
          service:
            type: ClusterIP
          ollama:
            gpu:
              enabled: false
              type: nvidia
              number: 0
            models: []
          runtimeClassName: ""
          persistentVolume:
            enabled: true
            size: 30Gi
          ingress:
            enabled: false
            annotations:
              swarm.io/exclude-from-evidence: "true"
        patches:
          - op: replace
            path: ollama.models
            value:
              - "{{ params.modelName }}"

          - op: replace
            path: ollama.gpu.enabled
            value: true
            when: "params.gpuEnabled == true"

          - op: replace
            path: ollama.gpu.number
            value: "{{ params.gpuCount }}"
            when: "params.gpuEnabled == true"

          - op: replace
            path: runtimeClassName
            value: nvidia
            when: "params.gpuEnabled == true"

          - op: replace
            path: persistentVolume.size
            value: "{{ params.storageSize }}"

          - op: replace
            path: ingress.enabled
            value: true
            when: "params.enableIngress == true"

          - op: add
            path: ingress.hosts.0.host
            value: "{{ params.apiHostname }}"
            when: "params.enableIngress == true"

  - name: open-webui
    dependsOn:
      - ollama
    source:
      type: helm
      repoUrl: https://open-webui.github.io/helm-charts
      chart: open-webui
      version: "5.20.0"
    images:
      - name: ghcr.io/open-webui/open-webui
        digest: sha256:2c26b46b68ffc68ff99b453c1d30413413422d706483bfa0f98a5e886266e7ae
    deployment:
      values:
        base:
          service:
            type: ClusterIP
          ollama:
            enabled: false
          extraEnvVars:
            - name: OLLAMA_BASE_URL
              value: "http://{{ components.ollama.release }}:11434"
            - name: ENABLE_OLLAMA_API
              value: "true"
          ingress:
            enabled: false
            annotations:
              swarm.io/exclude-from-evidence: "true"
        patches:
          - op: replace
            path: ingress.enabled
            value: true
            when: "params.enableIngress == true"

          - op: add
            path: ingress.hosts.0.host
            value: "{{ params.chatHostname }}"
            when: "params.enableIngress == true"

outputs:
  - id: chatUrl
    title: Chat UI
    type: url
    value: "https://{{ params.chatHostname }}"
    when: "params.enableIngress == true"

  - id: apiUrl
    title: API Endpoint (OpenAI-compatible)
    type: url
    value: "https://{{ params.apiHostname }}/v1"
    when: "params.enableIngress == true"

  - id: internalApi
    title: Internal API
    type: url
    value: "http://{{ components.ollama.release }}.{{ namespace }}.svc.cluster.local:11434"
```

### Application with Data Slots: RAG Agent

A complete `app.yaml` for a retrieval-augmented AI agent that consumes marketplace datasets through two slots: a required multi-dataset knowledge base and an optional single-dataset feedback store. The `corpus` slot is schema-constrained, so it matches (for example) the `retail-transactions` dataset above, which has a `text` column.

```yaml
apiVersion: swarm.cloud/v1alpha1
kind: AppDefinition

metadata:
  name: rag-agent
  title: RAG Agent
  description: Retrieval-augmented AI agent that answers questions over connected datasets.
  version: "1.2.0"
  category: ai-inference
  license: Apache-2.0
  tags:
    - ai
    - agent
    - rag

ui:
  sections:
    - id: model
      title: Model
      fields:
        - parameterId: modelName
          widget: select

parameters:
  - id: modelName
    type: string
    title: Model
    description: The LLM used for answer generation.
    required: true
    default: "llama3.1:8b"
    options:
      source:
        type: static
        values:
          - value: "llama3.1:8b"
            label: Llama 3.1 (8B)
          - value: "qwen2.5:7b"
            label: Qwen 2.5 (7B)

resources:
  min:
    cpu: "4"
    memory: "8Gi"
  recommended:
    cpu: "8"
    memory: "16Gi"

data:
  slots:
    - name: corpus
      title: Knowledge Base
      description: Datasets the agent retrieves answers from. At least one is required.
      required: true
      multiple: true
      accepts:
        - interface: clickhouse-native/v1
        - interface: s3-parquet/v1
      schema:
        tables:
          - columns:
              - name: text
                type: string

    - name: feedback
      title: Feedback Store
      description: Optional dataset with prior user feedback used to improve answers.
      accepts:
        - interface: clickhouse-native/v1

components:
  - name: agent
    source:
      type: helm
      repoUrl: https://charts.example.com/rag-agent
      chart: rag-agent
      version: "0.4.0"
    deployment:
      values:
        base:
          model: ""
          datasources:
            corpusJson: ""
        patches:
          - op: replace
            path: model
            value: "{{ params.modelName }}"

          - op: replace
            path: datasources.corpusJson
            value: "{{ data.corpus | json }}"

          - op: add
            path: datasources.feedback.url
            value: "clickhouse://{{ data.feedback.connection.host }}:{{ data.feedback.connection.port }}/{{ data.feedback.connection.database }}"
            when: "data.feedback.count > 0"

          - op: add
            path: datasources.feedback.password
            value: "{{ data.feedback.connection.password }}"
            when: "data.feedback.count > 0"

outputs:
  - id: agentApi
    title: Agent API
    type: url
    value: "http://{{ components.agent.release }}.{{ namespace }}.svc.cluster.local:8080"
```

---

## JSON Schemas

Formal JSON Schemas for all entities are located in the `specs/` directory:

| File                               | Description                                         |
|------------------------------------|-----------------------------------------------------|
| `app-definition.schema.json`       | Root schema for the complete `app.yaml` document.   |
| `component.schema.json`            | Schema for individual component definitions.         |
| `parameter.schema.json`            | Schema for individual parameter definitions.         |
| `value-source.schema.json`         | Schema for option/value source definitions.          |
| `deployment.schema.json`           | Schema for the deployment section (base + patches).  |
| `resource-requirements.schema.json`| Schema for resource requirement declarations.        |
| `data-interface.schema.json`       | Root schema for `DataInterface` documents.           |
| `data-definition.schema.json`      | Root schema for `DataDefinition` documents.          |
| `data-schema.schema.json`          | Shared semantic schema (tables and typed columns).   |
| `data-slot.schema.json`            | Schema for application data slot declarations.       |
| `publisher-secret.schema.json`     | Schema for publisher secret declarations.            |

All schemas use **JSON Schema Draft-07** and reference each other via `$ref`.

### Validation

To validate documents against the schemas:

```bash
# Using check-jsonschema
check-jsonschema --schemafile specs/app-definition.schema.json apps/ollama-webui/app.yaml
check-jsonschema --schemafile specs/data-interface.schema.json interfaces/clickhouse-native/v1.yaml
check-jsonschema --schemafile specs/data-definition.schema.json datasets/retail-transactions/data.yaml

# Using ajv-cli (register referenced schemas with -r)
ajv validate -s specs/app-definition.schema.json -r "specs/*.schema.json" -d apps/ollama-webui/app.yaml
```

---

## License

This specification is open source. See [LICENSE](LICENSE) for details.
