# Asset Structure Profiles
Copyright &copy; 2023, NVIDIA Corporation, version draft

> **NOTE** The usage of asset in this document should not be conflated with the `asset` attribute type. Asset in this context refers to an organizational structure of layers and prims.

## Background
USD provides a series of composition operators to assemble scenes from layers. Lacking a formal asset definition of its own, this allows USD to integrate with a variety of pipelines, versioning, storage, and formats.

Departments with different workflows and different asset structures can still be unified and presented as a single "stage" for downstream contexts like rendering.

This philosophy serves many use cases well. Simulations and renderers don't have to reason about the asset structure and don't have to be updated when that asset structure changes.

However, there are workflows around which "asset"-ness is critical, particularly in user ineterfaces. Consider direct manipulation in the viewport-- The asset is often the perfect granularity for authoring transform opinions. USD's first attempt at describing assets was through the `kind` metadata. Additional use cases brought along `ModelAPI`, `payloadAssetDependencies`, and the `assetInfo` dictionary.

These mostly function as annotations and while there are some rules, there isn't a lot of guidance as to how these should be utilized together in concert with the composition operators. A lack of clear guidelines can complicate introduction of new users and developers to the ecosystem. 

## Related Work

To address this issue, documents like the ASWF assets working group [guidelines](https://github.com/usd-wg/assets/blob/main/docs/asset-structure-guidelines.md) aim to provide a set of best practices. This is a great resource that can answer many questions for pipeline developers.

The document, however, is difficult to translate into a validation and repair pipelines. It's not clear under what circumstances (for example) failure to scope materials under a `mtl` scope should trigger an error.

Recent ASWF working groups have floated the idea of a standard set of variants and layer structures to aid development of certain tooling.

OpenUSD provides `usdchecker` which validates common axioms, mostly about schemas, but also flags missing references.

Interestingly, `usdchecker` provides a flag `--arkit` which adds additional rules to ensure `ARKit` specific compliance. This flag functions as an existance proof for this proposal-- demonstrating the need for different profiles of assets in the context of validating.

## Proposal

To improve the asset development and validation experience for  developers and users, the USD Ecosystem should introduce the concept of asset profiles and provide a playbook for how to develop these profiles.

There is no one correct way to structure an asset in USD, but there may be a correct way to structure an asset with a specific profile.

An asset profile contains a set of categorized rules. Rules should describe a rule violation's severity, detection, repairability, and exceptions. Rules should be as short and atomic as practical. Some overlap may be unavoidable. For example, a rule may provide a list of required variant sets, and then elaborate with additional rules for each required variant set. Profile rules should not aim to enforce schema rules but can enforce schema best practices relevant to the profile.

This proposal philosophically aligns with the C++ Core Guidelines.

### Identifiers
Rules should be identified via a hierarchy of UTF-8 Identifiers (similar to the restrictions placed on USD prim and property names).

The restriction to identifiers is the assumption that profiles will be used as command line arguments and embedded in other strings.

A rule could be uniquely identified via `StdAssetProfile/DefaultPrim/has_asset_name` where `StdAssetProfile` is the profile, `DefaultPrim` is the category, and `has_asset_name` is the rule.

> **NOTE** It make make sense to align unique identification of asset profile rules with an identifier scheme for schema validation rules.

### Severity
Each profile can define its own list of severities. Severities should be casefolded UTF-8 Identifiers.

Example (required?) severities:
> * **error**: Something in the pipeline is apt to break if not addressed. For example, a schema may have poor support in the pipeline and so even when used as specified, usage may be prohbited.
> * **warning**: The pipeline won't break, but the usage perhaps invites a lack of clarity or consistency and can be better structured. For example, naming conventions for prims.
> * **deprecated**: A feature may currently work, but may not work in the future. For example, usage of a `kind` that is going to be retired in the next pipeline release.
> * **performance**: Usage is correct and supported but may have unintended performance impacts.
> * **info**: A rule may just be informative, perhaps just declaritively noting that the profile has no opinion about a questions

### Brief Description
Rules should aim for a succint explanation that can easily be displayed as tooltips or command line output.

> **Example** The root prim must have component set to kind.

### Motivation
Rule documentation can elaborate on the motivations for a rule.
> **Example** The root prim must have its kind set to component so that tools know that the referenced asset should be treated as an atomic element when selecting, transforming, or deactivating.

### Detection
Even if it seems obvious, explain the preferred way to detect the error to stamp out ambiguities in the rule's description.

Both of these might be valid interpretations of a rule stating that a root node must have its kind set to component.
> **Example** Use `UsdModelAPI::IsKind` to validate `kind` instead of checking the `kind` metadata directly.

> **Example** Check the `kind` metadata is explicitly set to `component`, as the profile doesn't allow for non-core USD kinds.

It's okay if rules are not be automatically detectable.

> **Example** Prim names should not contain verbs

That is a reasonable rule to ask users to respect but might be hard to detect and repair.

### Repair
Explain if and when the rule can be repaired.
> **Example** If the root prim `kind` metadata is unset, simply set it to `component` in the asset's root layer. If `kind` is set, the asset cannot be repaired.

### Exceptions
Rules should expicitly note if there are any exceptions to the rule.
> **Example** Assets whose root layer has been tagged with `customLayerData = {bool test = 1}` are exempt from this rule due to some legacy pipeline test assets that are being actively retired.

### Profile Referencing
Profiles should be able to reference other profiles, profile categories, or invididual profile rules. This allows sites to refine or augment a set of general rules without having to redefine them and their validation.

## Usage
With a standard for describing profiles in hand, developers can start building tooling that nominally reference the profiles and their respective rules. Users can quickly refer to a document to understand the severity and reasoning of a `StdAssetProfile/DefaultPrim/has_asset_name` rule.

Managers and engineers working on leaf assets that don't necessarily require the full range of USD customization can lean on simple profile definitions to help boostrap their integration into the ecosystem. "Our tool produces `StdAssetProfile` compliant assets."

Teams in large organization can use profiles to describe what assets and features need to be considered for a given pipeline.

There is no one correct way to structure an asset in USD. There are many. By qualifiying the rules of successful asset structures will foster a more confident and collaborative USD ecosystem.

## Future Work

What's being proposed is entirely external to USD. It may make sense to develop an encoding of profiles (or pragmas to exclude rules) in the future once profiles are better established.

## Sample Profile
> ### DefaultPrim
> #### is_component
> The default prim of an asset must be of kind `component`
>
> **Severity**: Error
>
> **Motivation**
> This asset profile assembles geometry, materials, and skeletal rigging into a single atomic asset.
>
> **Detection**
> Use `UsdModelAPI::IsKind` to validate `kind` instead of  checking the `kind` metadata directly.
>
> **Repair**
> Override the type to `component` in the asset's root layer.
> 
> **Exceptions**
> None
> ### Organization
> #### gprims_under_geo
> All `Gprim`s must live under the default prim's `geo` scope.
>
> **Severity**: Error
>
> **Motivation**
> This keeps geometry, rigging data, and materials organized for the asset.
>
> **Detection**
> Find all defined gprims and check that their path starts with `{defaultPrim}/geo`
>
> **Repair**
> This cannot be automatically repaired.
> 
> **Exceptions**
> `Sphere`s in the `rig` scope are okay if and only if their purpose is set to `guide`.

## Questions 
* How should versions of profiles be handled?