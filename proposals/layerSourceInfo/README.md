# Layer Source Metadata
Copyright &copy; 2023, NVIDIA Corporation, version draft

## Overview
Developers and users want a standardized way to record and share information about the editor that produced the layer for tracking down (and in certain circumstances repairing) application specific defects and bugs.

## Related Work
### Version Control and Asset Publishing
Version control / asset publishing systems have robust tooling for tracking differences, users, and descriptions of changes. Pipeline written around version control systems can embed more detailed information about a layer's history and source context. As data generally becomes read-only during commit or publish events, the information is less likely to be stale. Layer source metadata should not aim to replace or replicate that toolchain.

### Telemetry
Telemetry tooling can record events and traces to help developers reason about processes and the environments in which they are running. This tooling can provide more introspectability and better analysis than layer metadata can provide.

### `documentation` field
It's worth noting that `UsdStage::Flatten` currently appends information about the flattening event to a layer's documentation string.

## Complicating Factors
The primary risk of recording source metadata in the layer is that it can become stale, misleading, or ambiguous under certain transformations (see [puzzles](#puzzles)). Additionally, due to dependence on external resources (asset resolvers, environment variables, etc.), it's impractical to record enough fields to guarantee reproducability. This metadata should not advertise itself as providing this.

 Fields scoped to layers cannot be preserved under stage flattening or other workflows which may merge or reorganize layers. A single prim may be composed of opinions from multiple versions of the same app (say modeled in Maya 2020 and animated in Maya 2022). As such, runtime behavior should avoid keying off these breadcrumbs.

## Proposal
To address user and developer needs for embedding application information while also minimizing the risk of introducing stale / misleading fields, this proposal recommends introducing `sourceInfo` layer metadata as a dictionary with a constrained set of fields.

> **NOTE** As breadcrumbs tied to a particular session of layer editing, fields encoded in `sourceInfo` should be considered ephemeral. Persistent data should be stored in other fields.

### Fields

There are four proposed fields (more detail in the API and examples below).
| key         | required | USD Type    | C++ Type |  description |
| ----------  | -------- | ----------- | ---------| -----------  |
| generator   | ✓        | `string`      | `std::string` | The name of the pipeline, application, or script which last generated or edited the layer
| versions    | ✓        | `dictionary`  | `std::map<std::string, std::string>` | Sparse list of relevant library, application, or pipeline versions that contributed to the formation of the layer   |
| flags       | ✓        | `dictionary`  | `std::map<std::string, std::variant<int, bool>>` | Sparse list of relevant library, application, or pipeline feature flags that contributed to the formulation of the layer. The last element of the hierarchy identifier is the flag name. This should not be a reflection of the environment or `TfEnvSettings`.|
| fixes     | ✓        | `string[]`    | `std::vector<std::string>` | An ordered set of unique strings recording breadcrumbs about transformations, repairs, or fixes that may have been applied to a layer after it was written by the editor. Fixes are the only field that may be updated. Authored fixes may not necessarily use the library versions and flags specified. Ideally, editors always are up to spec and never require fixes so this field is always empty.

### Scope
This proposal argues for encoding this metadata with a small number of easily understandable fields presented clearly as breadcrumbs. When considering if this proposal should be augmented with additional data, we must consider
1. If this data is better managed and queried for users through telemetry and version control systems
2. If including this field helps or hinders the interchange and assembly of data across space and time through USD

Applications and pipelines should be cautious about including information like user name, machine name, project name, etc. when exporting. It's worth considering that search paths and file paths may indirectly contain this information.

### Hierarachical Identifiers
The dictionary keys and strings are mostly unconstrained with two exceptions.
* They should not contain tabs or new lines
* `/` implies a hierarchy, where each element in the hierarchy may (but may not) have its own set of flags and versions

Examples:
* `Node Graph Editor` : Refers to an application for editing USD node graphs
* `PublisherABC ToolsetXYZ/plugin.so` : Refers to a plugin for a toolset made by PublisherABC
* `Project Code Name/make_asset`: Refers to a script associated with a particular project
* `Asset Publish Pipeline/Finalize Geometry Layer`: Refers to a stage in a pipeline

Each element in the hierarchy may have their own set of versions, flags, or fixes that can be assumed to be relevant to their descendants.  In above examples, `"PublisherABC ToolsetXYZ"` is a single versioned element, which is why the publisher isn't a part of the hierarchy (ie. prefer `"PublisherABC ToolsetXYZ"` to `"PublisherABC/ToolsetXYZ"`).

The primary need these fields addresses is the interchange of OpenUSD layers between users and developers. Encoding the version of various libraries, plugins, and applications can help provide context as to where bugs or issues may lie.

### Examples
This example demonstrates an application export where the export process not only records its own version but also the version of key libraries like python and the OpenUSD repository.

```
sourceInfo = {
    string generator = "Publisher Toolset/UsdExport.so"
    dictionary flags = {
        int "OpenUSD/TF_UTF8_ENABLED" = 1
    }
    string[] fixes = []
    dictionary versions = {
        string OpenUSD = "0.2023.5"
        string "Publisher Toolset" = "v5.1"
        string python = "3.7"
        string "Publisher Toolset/UsdExport.so" = "beta"
    }
}
```

This demonstrates how a pipeline script might record information. It demonstrates flags being used to record information about the OpenUSD runtime configuration as well as fixes applied to the layer after its initial generation.

```
sourceInfo = {
    string generator = "Studio XYZ Pipeline/Flatten Geometry"
    dictionary flags = {
        int "OpenUSD/TF_UTF8_ENABLED" = 0
        bool "Studio XYZ Pipeline/Experimental Curve Support" = true
    }
    string[] fixes = ["fixed_inverted_normals_v2"]
    dictionary versions = {
        string OpenUSD = "0.2023.5"
        string "Studio XYZ Pipeline" = "studio_xyz.2022.5"
    }
}
```

This demonstrates how a layer produced through the flattening API might be recorded in the `sourceInfo`.
```
sourceInfo = {
    string generator = "UsdStage::Flatten"
    dictionary flags = {}
    string[] fixes = []
    dictionary versions = {
        string OpenUSD = "0.2023.5"
    }
}
```

An earlier version of this proposal considered whether or not a timestamp should be encoded as a unique field. This seems like it might be the most potentially misleading field and redundant with information version control, asset publishing, and the filesystem could provide. This information can be retrieved with `ArResolver::GetModificationTimestamp`.

The interfaces for `sourceInfo` should get and set all fields atomically to minimize creating a mixed set of fields from different 'Save' events.

### API
Source metadata should be authored in concert with a 'Save' event. There should no be independent `Set` API. OpenUSD should provide either in core or through `UsdUtils` the ability to save layers and stages with the following options.
* Clear all `sourceInfo` metadata
* Clearing and replacing all `sourceInfo` metadata with an updated dictionary. `fixes` should be empty in this specification. `flags` should be sufficient to encode all application specific options.
* Preserving `sourceInfo` metadata, appending a user provided string to the `fixes` field.

The information stored in the `sourceInfo` field is determined entirely by the generating context. OpenUSD should not record or replace information by default.

### Validators, Linters, and Fixes
This proposal discourages keying runtime behavior off of the breadcrumbs (ie. flip the normals because the file came from "Node Graph Editor X version 2022.5"). However, if the content exported by a particular version of a tool is known to be problematic, fixes to layers still need to be made and querying the `sourceInfo` field may be the most practical way of repairing data.

This proposal advocates under that scenario that it's best to have validation or linting scripts apply fixes to the layer. Ideally, all fixes applied are idempotent-- the fix when applied N times to the layer is the same as the fix applied once. Validators and linters should strive for that ideal. However, there may be cases (like flipping normals or storing radii instead of diameters as point widths) where it cannot be divined that a fix has already been applied. Validators and linters in that case should record these fixes in the `fixes` field.

A validator run may result in multiple entries in the `fixes` list but entries should generally not be prim specific. (ie. `fixed_inverted_normals_v2` can be assumed to be applied to all specs in the layer vs. `fixed_inverted_normals_v2 on /root/parent/mesh_2`).

## Puzzles
### Fix with a different version of USD
Consider "Site A" injesting data from "Contractor B" for use in production. "Contractor B" produced the layer using "Node Editor X" which recorded the "OpenUSD" version as `2022.5`. "Site A" is using version `2022.11` of "OpenUSD" and runs a script on injest that adds "Injested on 1/2/2023 from Contractor B" to the layer's comments. "Site A" was not aware that `2022.11` bumped the crate version from `0.8.0` to `0.9.0`. On save, the produced crate file is no longer openable in tools using `2022.5` which was not aware of the new crate version. The layers are now in a puzzling but certainly not an invalid state. The layer is properly reporting the correct crate version. It wouldn't be clarifying for the script to update the versions or flags associated with the "Node Editor X"'s process (Why? The puzzle has just been moved-- there may be no published version of "Node Editor X" that uses `2022.11`.) At the same time, it's likely not desirable for the injestion script to clear the editor information field. Depending on how core `sourceInfo` info becomes, the stage could warn on save if there is a version mismatch with the "OpenUSD" version string. However, one could construct a similar scenario for flags that could produce similarly puzzling results.

This puzzle should serve as another reminder that it's important that applications do not encode runtime behavior based on the `sourceInfo` metadata.

### Editing a Layer With Multiple Source Editors
When USD is used as source data, it may be used as a neutral format between tools. Consider "Tool ABC" which generates inverted normals for left handed geometry. "Modeler 123" generates correct normals. Meshes generated in "Modeler 123" may be edited in "Tool ABC", but since the originator of the normals had generated them correctly, "Tool ABC" doesn't corrupt them. A validator or linter sent to fix the normals of meshes generated by "Tool ABC" cannot divine that the normals were correctly generated by another application and were simply passed through "Tool ABC"

This is another example of why idempotence such be paramount to any validator pipeline. Keying off of the `sourceInfo` breadcrumb may be misleading without broader pipeline context and should always be a last resort. When a validator has no choice but to consult version, layers and assets may still need to be spot checked for correctness.

## Alternative Formulations
### Single String
Instead of utilizing USD's dictionary type, let users encode application data in a single string.
```
string sourceInfo = "Publisher ToolsetXYZ 1.2/{"libraries" : {"OpenUSD": "0.2023.5"}}
```
The single string encoding could be helpful in reinforcing the ephemeral and sparse nature of these breadcrumbs. However, it could also lead to parse errors in validation scripts and user interface presentation.

### History
Instead of storing the single latest editing application as the `sourceInfo`, store a the history that contributed to the layers formulation. `fixes` could likely go away in this formulation as the fixing tool would just be another editor. There a lot of questions to answer with this formulation, and it may be redundant with version control systems. Some questions to consider-- Are applications expected to record every save event? Or just when the latest editor name changes? Do validators attempt to reason about history? When do you clear history?

### `flags` as `dictionary`s
To make per library / application flags explicit, encode `flags` as a dictionary of dictionaries.
```
flags = {dictionary Application = {bool app_flag = true}
         dictionary "Application/UsdExport.so" = {bool plugin_flag = true}}
```
Using `VtDictionary` as the type of flags would make extraction and authoring of flags fairly straight forward and consistent with other fields like `assetInfo` and `customData`.

The `Application/plugin/flag` format was picked in the primariy proposal as it's easier to validate / decompose the contents of each flag are simple fields. The STL (`std::lower_bound`) can be used to easily filter flags that are associated with a particular identifier. This lookup can be wrapped up in `sourceInfo` specific API without relying on the `VtDictionary` recursive lookup syntax. While it's unlikely that `Application` or `plugin` would have `/` as an identifier (as that would complicate their filesystem representation), it's not clear if that same logic holds for application flags. The purpose of flags is to record a handful of important feature flags per library; it's not expected to store a hierarchy of user preferences.
 
## Questions
* Are there fields which are privacy concerns when publishing / externalizing layers? Does including `sourceInfo` make OpenUSD more complicated to use as users have to consider removing data applications add to their files?
* Should file format plugins populate the `sourceInfo` field for `usdcat` contexts?
* However well reasoned this set of fields might be, will application developers ultimately be contrained by this and introduce application specific extensions as `customLayerData` that can get out of sync with `sourceInfo`?
* Flags are restricted to be `int` or `bool` types. Should they just be `int`? Should they support other / arbitrary types?
* Should this proposal aim to use any existing standards?
* Should this proposal publish a list of canonical identifiers? (ie. `OpenUSD`, `Operating System`, `python`, etc.)