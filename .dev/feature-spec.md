# tidytree Feature Specification Notes

## Purpose

This document records candidate features that would strengthen `tidytree` as
the core data manipulation layer for the `ggtree` ecosystem. The focus is on
APIs that make tree structure, annotation, and performance-sensitive traversal
easier to use from tidy workflows while remaining compatible with `phylo`,
`tbl_tree`, and `treedata`.

## Design Principles

- Prefer small, composable functions over monolithic workflows.
- Keep `phylo`, `tbl_tree`, and `treedata` behavior aligned where possible.
- Return tidy tabular outputs for inspection APIs and tree-like objects for
  transformation APIs.
- Avoid requiring users to repeatedly rebuild traversal state for large trees.
- Provide structured diagnostics that can be tested programmatically.

## 1. Structural Validation

### Problem

Tree conversion and manipulation functions currently assume valid structures in
many places. Invalid inputs can fail with low-level indexing errors or warnings
that are difficult to act on.

### Proposed API

```r
validate_tree(x, ..., strict = TRUE)
validate_tbl_tree(x, ..., strict = TRUE)
is_valid_tree(x, ...)
```

### Return Value

`validate_tree()` should return a tibble with one row per issue:

```r
tibble::tibble(
    check = character(),
    severity = factor(levels = c("error", "warning", "info")),
    node = list(),
    message = character()
)
```

`is_valid_tree()` should return a scalar logical and optionally attach the
diagnostic tibble as an attribute.

### Checks

- Required columns for `tbl_tree`: `parent`, `node`, `label`.
- Exactly one root for rooted tree-like structures.
- No duplicated child node in edge matrix.
- No node with multiple parents.
- No disconnected nodes.
- Node IDs are compatible with `phylo` conventions: tips first, internal nodes
  after tips.
- `edge.length`, `tip.label`, and `node.label` lengths match topology.
- Annotation rows in `treedata@data` and `treedata@extraInfo` reference valid
  node IDs.

### Implementation Notes

- Reuse the internal logic currently spread across `valid.tbl_tree()`,
  `valid.tbl_tree2()`, `valid.edge()`, `rootnode()`, and accessor methods.
- Keep validation side-effect free. Avoid `cli_alert_*()` in core validators;
  format messages at the caller boundary.
- Add methods for `phylo`, `tbl_tree`, and `treedata`.

## 2. Tree Comparison and Diff

### Problem

Users often need to confirm that a data operation changed only annotations, not
topology. Today this requires custom comparisons of edges, labels, and clades.

### Proposed API

```r
compare_tree(x, y, ..., by = c("node", "label", "clade"))
same_topology(x, y, ...)
diff_tree(x, y, ...)
```

### Return Value

`compare_tree()` should return a structured list:

```r
list(
    same_topology = TRUE,
    node_map = tibble::tibble(node.x = integer(), node.y = integer()),
    edge_diff = tibble::tibble(),
    label_diff = tibble::tibble(),
    annotation_diff = tibble::tibble()
)
```

### Comparison Modes

- `by = "node"`: compare using current node IDs.
- `by = "label"`: map tips and internal labels where available.
- `by = "clade"`: compare internal nodes by descendant tip sets.

### Implementation Notes

- For clade comparison, use sorted descendant tip labels as stable clade keys.
- For large trees, avoid constructing large pasted strings for every clade when
  possible. A future hash-based representation can be added later.
- Keep annotation comparison optional because list-columns and duplicated joins
  may need package-specific rules.

## 3. Path and Distance APIs

### Problem

Common tasks such as finding a path to root, a path between two nodes, or node
depth require users to combine `ancestor()`, `parent()`, `offspring()`, and
manual joins.

### Proposed API

```r
path_to_root(x, node, ...)
path_between(x, node1, node2, ...)
node_depth(x, unit = c("edge", "branch.length"), ...)
edge_path(x, node1, node2, ...)
```

### Return Values

`path_to_root()`:

```r
tibble::tibble(
    node = integer(),
    parent = integer(),
    label = character(),
    depth = integer(),
    branch.length = numeric()
)
```

`path_between()` should return an ordered tibble from `node1` to `node2`, with
the MRCA marked by a logical column.

### Implementation Notes

- Use a parent vector indexed by node ID for `phylo`.
- For `tbl_tree`, use `parent` and `node` columns directly.
- `unit = "edge"` counts edges; `unit = "branch.length"` sums branch lengths.
- Handle missing branch lengths explicitly and fall back only when documented.

## 4. Clade and Subtree Operations

### Problem

Subtree extraction and clade inspection are core tree data tasks. Existing
functions cover parts of this space, but the naming and return types can be
more consistent.

### Proposed API

```r
clade_nodes(x, node, self_include = TRUE, ...)
clade_tips(x, node, ...)
extract_subtree(x, node, ...)
collapse_clade_data(x, node, fun = dplyr::summarise, ...)
```

### Behavior

- `clade_nodes()` returns all descendant nodes plus the focal node by default.
- `clade_tips()` returns descendant tip rows or tip IDs depending on input type.
- `extract_subtree()` returns the same object family as input where feasible.
- `collapse_clade_data()` should summarize annotations for collapsed clades
  without requiring plotting code.

### Implementation Notes

- Build on `offspring()` and `tree_subset()`, but keep return semantics simpler.
- Document how root edges and branch lengths are handled during extraction.

## 5. Annotation Conflict Utilities

### Problem

Join operations can create duplicated nodes, nested list-columns, and suffix
conflicts. This is sometimes desired, but users need explicit helpers to inspect
and resolve the result.

### Proposed API

```r
annotation_conflicts(x, ...)
resolve_annotation(x, columns, strategy = c("first", "last", "list", "error"), ...)
unnest_annotation(x, columns, ...)
summarise_annotation(x, by = "node", ...)
```

### Use Cases

- Detect duplicated annotation rows after joins.
- Convert list-columns created by repeated node annotations into scalar values.
- Fail early when a join would duplicate tree rows unexpectedly.

### Implementation Notes

- Reuse `.internal_nest()` and `.check_duplicated_rows()` where appropriate.
- Prefer explicit user choice over silent deduplication for new APIs.

## 6. Traversal Index for Large Trees

### Problem

Repeated calls to `ancestor()`, `offspring()`, `MRCA()`, and related operations
can rebuild the same parent/child relationships many times. This is expensive
for large trees.

### Proposed API

```r
tree_index(x, ...)
ancestor(index, node, ...)
offspring(index, node, type = c("all", "children", "tips", "internal"), ...)
MRCA(index, node1, node2 = NULL, ...)
```

### Object Structure

```r
structure(
    list(
        parent = integer(),
        children = list(),
        tip = logical(),
        label = character(),
        root = integer(),
        edge_length = numeric()
    ),
    class = "tidytree_index"
)
```

### Implementation Notes

- `parent` should be an integer vector indexed by node ID.
- `children` can be a list keyed by node ID or an adjacency structure using
  edge offsets.
- Use this index internally for high-level traversal functions when available.
- Keep the index invalidation story simple: it represents a snapshot, not a
  live view of a mutable tree.

### Performance Targets

- Build index in O(n).
- `parent(index, node)` in O(1).
- `ancestor(index, node)` in O(tree height).
- `offspring(index, node)` in O(size of subtree).
- `MRCA(index, node1, node2)` in O(height) for two nodes, with room for future
  batch optimizations.

## 7. Tip/Internal-Specific Verbs

### Problem

Many tidy operations need to target only tips or only internal nodes. Users
currently repeat `isTip` filters or derive them manually.

### Proposed API

```r
filter_tip(.data, ...)
filter_internal(.data, ...)
mutate_tip(.data, ...)
mutate_internal(.data, ...)
select_annotation(.data, ...)
```

### Behavior

- For `tbl_tree`, return a `tbl_tree` when the topology columns remain valid.
- For `treedata`, preserve the tree and modify annotation slots where possible.
- For `phylo`, convert to `treedata` or `tbl_tree` only when documented.

### Implementation Notes

- These should be thin wrappers over existing dplyr methods.
- Avoid surprising topology changes. Filtering tips should not silently drop
  tree structure unless the function name explicitly says so.

## Suggested Implementation Order

1. Structural validation, because it improves error messages and protects later
   APIs.
2. Traversal index, because path, clade, and performance-sensitive APIs can
   reuse it.
3. Path and distance APIs, because they are broadly useful and easy to test.
4. Clade/subtree operations, building on traversal helpers.
5. Annotation conflict utilities.
6. Tree comparison and diff.
7. Tip/internal-specific tidy verbs.

## Testing Strategy

- Add shared fixtures for bifurcating and multifurcating trees.
- Test every new API on `phylo`, `tbl_tree`, and `treedata` where supported.
- Include malformed tree fixtures for validation errors.
- Include named and unnamed internal nodes.
- Include branch lengths, no branch lengths, and root edge cases.
- Add performance regression tests only as opt-in benchmarks, not as CRAN tests.

## Open Questions

- Should validation return a tibble only, or a richer S3 object with print
  methods?
- Should `tree_index()` be exported immediately, or used internally first?
- Should path APIs return node IDs only by default, or full `tbl_tree` rows?
- How strict should topology preservation be for tip/internal-specific verbs?
- Should annotation conflict resolution live in `tidytree` or a higher-level
  ecosystem package?
