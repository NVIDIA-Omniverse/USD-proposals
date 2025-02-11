# Listless Sparse Array Edits
Copyright © 2025, NVIDIA Corporation

## Overview
This document provides a formulation of sparse array edits that is closed,
associative, and does not require storing sequences of edits by
constraining edit operations to `replace` and `append`.

This proposal focus on these two operations because
* Neither trigger "reindexing"
* They cover many proposed usages in the original proposal

## Reindexing
The original proposal introduces "edit sequences as a value type" to 
provide associative and closed partial composition for all specified
operations. Insertion and deletion affect the indices of their rightward 
neighbors. Reindexing (generally) requires history to maintain
associativity. Weaker and stronger operations can change the underlying
indices which means that the combined value needs to preserve the order
in which the operations were applied.

## Coordinated Schema Indexing
A primary concern of this proposal is that while sequences of edits solve
the internal composition problems introduced by reindexing, they are not
able to apply similar corrections to index-based schemas.

Most geometry types and primvars specify index buffers. If data is 
inserted or removed, a sibling property must be reindexed.

```
def PointInstancer "instancer" {
    points3f[] positions = {prepend:  [(300, 300, 300)]}
    ...
    color3f[] primvars:color = {prepend: [(1, 0, 0)]}
    # an inserted element triggers a reindex of all rightward values
    int[] primvars:color:index = [0, 1, 1, 1, 1, 2, 2, 2, ...]
}
```
The complexities of proper reindexing management could discourage users and
tools from robustly supporting array edits.

## Proposal
It might be observed that a similar transform could likely have been
achieved through `append` without triggering a reindex. Let's restrict the
set of operations to those that don't result in reindexes.

Define `E` an edit as a value with two subfields `append` and `replace`.

```
E = (
    # replace is a sparse map of indices and
    # replacement values 
    replace: {},
    # append is an array of additional values
    append: []
)
```

```
E = (replace: {1: "hello"},
     append: ["world"])
```

Define combination of two sparse edits `Es` and `Ew` as.
```
combine(Es, Ew) = (
    replace: merge(Es.replace, Ew.replace),
    append: Ew.append + Es.append
)
```

```
combine(
    (
        replace: {0: "hello"},
        append: ["world"],
    ),
    (
        replace: {0: "ciao"},
        append: ["again"]
    )
) => (replace: {0: "hello"},
      append: ["again", "world"])
```

In most cases, combining an edit `E` with a dense value `D` can result in a new
dense value.

```
combine(E, D) = merge(E.replace, D + E.append)
```


```
combine(
    (
        replace: {0: "hello"},
        append: ["world"],
    ),
    ["again"]
) => ["hello", "world"]
```
### Sepculative Replaces
However, there is an edge case to consider: if a replace depends on a stronger
append.


```
combine(
    (   
        append: ["world"]
    ),
    (
        replace: {0: "hello",
                  1: "again"},
    ),
    ["ciao"]
) => ???
```

If you evaluate the sparse ops first, they produce a new sparse op
```
    (   
        append: ["world"],
        replace: {0: "hello",
                  1: "again"},
    )
```
which gets reolved to `["ciao", "world"]` and then each element is replaced to `["hello", "again"]`.

If you evaluate the middle sparse and dense op first, `["caio"]` gets replaced
with `["hello"]` but there's no place to hold the `["again"]`.

To preserve associativity, this formulation proposes that a "speculative replace"
results in a "dense" or "blocking" edit.

```
combine(
    (
        replace: {0: "hello",
                  1: "again"},
    ),
    ["ciao"]
) => (
        replace: {1: "again"},
        append: ["hello"],
        blocking: True
     )
```

Any dense array is equivalent to a blocking version of an append only edit.

```
[e0, e1, e2, ...] == (append: [e0, e1, e2, ...], blocking = True)
```

Speculative replaces are expected to be rare, and most edits over dense
arrays in practice will result in dense arrays. It may even make sense for validation to
discourage them. We primarily describe them for closed associativity of the
set, but there may be practical usages that we had not considered.

## List Operations as Value Types
The above formulation would address many use cases for sparse edits, but it's worth considering the ideal formulation of removal for uniform attributes like `xformOpOrder`.

This case stores tokens that are expected to be unique and would benefit from appending, prepending, and *removal by value* not index. A user likely is not interested in deleting the 6th value in the array as much as they are expressing the intent to remove `xformOp:scale:tempHack` from the chain of ops.

Given that `dictionary` types are being considered for promotion to attribute value types, it's worth considering whether listops should be considered as well, at least for uniform attributes.

```
uniform prepend tokenlistop xformOpOrder = 
["xformOp:translateXYZ:offset"]
```

While the cost of changing a core schema like this is high, removing by index may not address the underlying user intent as well.

## Additional Sparse Edit Types 
Array edits could be extended to support additional edits as long as
they adhere to the principle of operating in the same index space.
However, this proposal recommends focusing exclusively on robust support
for the common `append` and `replace` use cases.

## Summary
This proposal advocates a limited set of index based array
operations that avoid the composition and schema-space complexities
reindexing introduces. Promoting the already successful listop as a
new attribute value type can then cover cases that cannot be handled
by arrays.

This gives users and developers two distinct paths when designing
their data strutures, schemas, and tooling: value based list
operations and index based array edits.