# Retained Variations

Copyright &copy; 2023, NVIDIA Corporation, version 1.0

## Background
Variant sets are a powerful composition arc that allow asset creators and pipelines to build high level interfaces to prim hierarchies. Any opinion (including other arcs, even other variant sets) can be expressed in the context of a variant of a prim hierarchy. When that variant is "selected", its opinions are composed (per LIVRPS). The opinions of the unselected variants are ignored.

While unselected variant opinions can be expressed and queried at the layer level, there is no mechanism for inspecting the composed result of other variations than the current selection.

## Motivation
### Preloading Variant Selections
Interactive applications may need to keep a set of level of detail variants warm to enable fast variant switching. There's a need to preload and retain certain variant combinations in order to have valid prim hierarchies for caching. Applications may want to apply validatation to variants to ensure they are safe for selection. These background variations need resync notices as if the variant was selected to keep caches up to date.

### Fast Loading of Stages
While the stage is being fully composed and processed for rendering, it might be useful to present decimated versions of models and materials that are cheaper to compose and prepare for the the renderer. There are workflows which attempt to address this (like `UsdGeomModelAPI` cards and `purpose` workflows). This would be a more generalized way to express standins across domains using the full gammet of scene description while defering composition of the (presumably) more expensive full render hierarchy.

### Variant User Interfaces
User interfaces for editing and viewing variant opinions may want to edit and display multiple variations simulatenously without selection.

For example, consider an interface for setting albedo values on material prims.

| color_variant   | red        | blue              | green          |
|-----------------|------------|-------------------|----------------|
| `inputs:albedo` | `(0.6, 0.0, 0.0)` | `(0.0, 0.0, 0.6)`   | `(0.0, 0.6, 0.0)` |

Users would like composed prims available for each variant to accurately reflect USD's value resolution. Users would also need to be able to map variations to edit targets for these prims.

### Pipeline Validation and Analysis
A pipeline engineer may like to non-destructively load variants to identify potential dependent layers if variant selections were to change downstream. They may like to non-destructively load variants to identify potential errors were the variant selections to change.

## Alternative Approaches
### Shadow Stages
With the current API, keep multiple shadow versions of the stage open, one per variant. In the preloading variant selections example, perhaps there's a "high", "low", and "medium" stage. Layers are shared between stages so that edits affect all levels of detail and resync notices are properly emitted. A multiplexing library would be used to negotiate which prim hierarchy is rendered from which composed stage.

This structure is limiting and hard to make dynamic, and requires one stage per variant. Performance wise, there's no ability to share composition results (say from instancing) across stages. This structure would be possible but hard to make work for a variant editor without an API on top for managing and generating the shadow stages.

### Tagging
With the current API, tagging mechanisms like `purpose` can be leveraged to filter out prims based on context. Schemas could be developed to act as switchboards for the hierarchy. However, that means having all versions loaded in the scene graph without the ability to share opinions between versions (without careful scene graph construction and leveraging inherited properties).

### Educated Guessing
Certain workflows may know enough about their opinions and asset structures to effectively guess at what the composed result would be. However, this could be prone to bugs or unexpected behavior in general.

## Proposal
### `__Variation` Hierarchies
Take inspiration from instancing and allow for private hierarchies to live on the stage. Just like `__Prototype` hierarchies they would not be writable. However, just like `__Prototype` hierarchies can be edited through any associated `inherit`s arcs, these alternate hierarchies could sometimes be edited through conversion to the appropriate `EditTarget`s for their respective variant sets.

Let's use the term **variation** to refer to these alternative hierarchies. Each variation is associated with one or more variant selections. The variant selections in a variation should always be considered "strongest" and are not associated with any layer. The goal of a variation is to compose the result as _if_ variant selection had resolved to a particular set of values, and _not if_ a particular set of layer edits were to be made. Only the user and application can determine if such an edit is authorable given the editable layer structure.

### The "Variant Interface"
Any field in USD may be overridden by a stronger layer, but there's a recognition that downstream contexts need to be careful with what fields get overrides. Field updates may need to be coordinated (such as points and their topology). Referenced asset hierarchy changes could result in overrides falling off.

Variants flip this contract and put the burden on the asset maintainer. If a user has taken the time to setup a `color_variant`, it's done to advertise to downstream contexts that the necessary primvars and material edits have been coordinated to produce a `red` or a `green` version of a particular asset. If the underlying material network changes or the geometry changes, it's on the asset maintainer to ensure the variant interfaces are still valid.

It's the asset interface aspect of variants is what necessitates this additional feature.

### Anchoring
Every variation is **anchored** to another prim. Retention is limited to the lifetime of that prim. If the anchoring prim expires, the variations are necessarily forcibily released. (Variations can be manually released.)

Anchoring prims are generally in the primary composed scene graph, but may be prims in other variation hierarchies or even in instance prototypes.

### API
The API should allow for variations to be "retained" until they are explicitly "released". Composition can be accelerated when variant "selection" resolves to the retained variant. Promoting a retained variation as the selected variation should not change its retained status. Retaining and releasing variations should have similar APIs and rules as loading and unloading payloads. _Retained variations do not persist as recorded layer state._

The completed retained variant API should provide
* Explicit release / retain methods at the prim level
* A stage level release (a reset)
* The ability access variation `UsdObject`s from the anchoring `UsdObject` and vice versa

#### Example: Inspecting variations of properties
```python
variations = [
    prim.RetainVariation({"color_variation" : v})
    for v in ["red", "blue", "green"]
]

paintMaterial = prim.GetPrimAtPath("/asset/materials/paint")
albedo = paintMaterial.GetInput("albedo")

print(f"Here are all the potential values of {albedo}: "
      f"{[v.GetObject(albedo).Get() for v in variations]}")
prim.ReleaseAllVariations()
```

### Nesting Retained Variants
Just like support for nested instancing can be complicated for clients, supporting nested retained variants can be similarly complicated (and potentially _expensive_ as permeatations multiply). Just like users need to consider where and when to leverage instancing, users need to consider where and when to leverage retained variants. **Every prim shouldn't be an instance. Every variant shouldn't be automatically "retained".**

### LIVRPS
At all times, while the selections in a variation are considered the strongest opinions, LIVRPS must be respected for all other fields (including variant selections not specified as a variation).

### Instancing
Prototype descendants may have variations retained. This may be surprising at first, but consider an instanced "Car" model with a "CarPaint" material that has a few color variants. While instanced, a user can use `inherits` arcs to change the color variant of the material. As such, there may be valid preview or preload workflows. 

If a prototypes are regenerated due to instance key changes, retained variations associated with the prototype may not persist.

### Selection and Notices
If a retained variation becomes effectively selected on the anchoring prim, the retained variation hierarchy persists. Notices must be observed as coming from the retained variation or the selected variation. Some observers (like a renderer leveraging retained level of detail variations) can immediately swap in the precomputed variation.

If the anchoring prim is backed by a released retained variation, that's okay. The anchoring prim now behaves as if it was never backed by a retained variation.

### Relationships and Collections
Resolving relationship targets and collections that cross the variation boundary are complicated by this. API may need to be written to let relationships targets resolve against retained variations.

It also may be not strictly necessary for an MVP. Being able to just preload geometry could still be a huge optimization even if evaluating targets like material bindings are deferred.

### Ancestral Traversal and Inheritance
A lot of tooling in USD relies on querying / traversing parent hierarchies (like object to world transforms). API may need to be written (perhaps something akin to `UsdTraverseInstanceProxies` to allow computations to traverse the ancestors of the anchoring prims).

### Out of Scope
#### Blending
This is not intended to support workflows like blend shapes or other rigging targets. Choosing to retain a variant is strictly a runtime decision for variant editing workflows and performance reasons.

## Summary
The ability to retain variations is an ambitious change to USD. However, scene graph instancing has already blazed a trail in the space of restricted prims living outside of the true scene hierarchy and demonstrated their benefits.

Retaining variations on the stage does not change the fact that there is one true selection state. It's strictly a feature of the runtime for interfaces to _preview_ other selection states for authoring, inspection, and preloading.

## Questions
* One could imagine that background loading of payloads being a useful concept as well. Should this proposal be reimagined as a more general `__Retained` concept?
* Could the retained concept be entirely unified with prototypes? Are these really "Variation Prototypes"?
* This document argues that the interface aspect of variants makes this arc special to warrant this feature. One could argue that dynamic payloads also have this interface property. Could dynamic payloads be considered for retention?
* Can retained variants (and potentially payloads) be implemented in a way that the primary scene graph retains read thread safety while computing?