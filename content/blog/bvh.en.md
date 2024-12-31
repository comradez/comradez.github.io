+++
title = "Rendering Notes: BVH"
date = "2022-11-21 22:52:49"
authors = ["茨月"]
+++

This blog post discusses the design, construction, and optimization of BVH (Bounding Volume Hierarchy), a commonly used acceleration structure in ray tracing. The content is relatively basic and does not include cutting-edge algorithms.

<!-- more -->

Recently, I decided to fill in some gaps in my knowledge of rendering, so I revisited pbrt (Physically Based Rendering Techniques). I realized that my knowledge gaps were larger than I thought... This time, I’m focusing on BVH notes because, in my personal renderer, I’ve only ever implemented KD-Tree and never BVH, let alone optimizations. So, I’m taking this opportunity to relearn this more commonly used acceleration structure in practical applications.

## What is BVH?

**Bounding Volume Hierarchy**, translated as "层次包围盒" in Chinese, is another tree-like data structure used to accelerate ray intersection, alongside KD-Tree. The key difference between BVH and KD-Tree is that BVH partitions objects, while KD-Tree partitions space.

## Construction of BVH

The construction of BVH is straightforward: First, we compute the Axis-Aligned Bounding Box (AABB) for each object in the scene. Then, we work bottom-up, merging two existing AABBs into a new, minimal AABB that encloses both, and place it on the tree as a node. This process continues until we obtain a global AABB that encloses all objects.

<div class="side-by-side-container"><img id="should-invert" src="https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/Primitives%20and%20hierarchy.svg" style="width: 60%"/></div>

In practice, we start top-down, recursively dividing the entire set of objects in the scene, selecting a dimension for binary partitioning each time, and then work bottom-up to determine the current bounding box.

This raises a question: **How should we partition?**

### Basic Approaches: Midpoint Partition and Equal Counts Partition

The simplest partitioning methods are [Midpoint Partition](https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L238-L251) and [Equal Counts Partition](https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L252-L262).

First, we find the bounding box formed by the midpoints of all existing bounding boxes and identify the dimension with the largest span as the partitioning dimension. Midpoint Partition selects the midpoint of this interval, grouping objects on the left and right sides. Equal Counts Partition ensures that the number of objects on both sides is equal. Both are basic partitioning methods.

### Advanced Approach: Surface Area Heuristic (SAH)

While the above methods are simple, they are not very effective. In practice, the [Surface Area Heuristic](https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/Bounding_Volume_Hierarchies#TheSurfaceAreaHeuristic) (SAH) is more commonly used. This algorithm is based on the following idea:

At each step of BVH construction, we can choose to either create a leaf node or partition the objects further. If we denote the intersection cost of the \\(i\\)-th object as \\(t_{\text{sect}}(i)\\), then the intersection cost of creating a leaf node is:
$
\sum_{i=1}^Nt_{\text{sect}}(i)
$
where \\(N\\) is the number of objects. If we partition the objects into two groups, A and B, the intersection cost becomes:
$
c(A, B) = t_{\text{trav}} + p_A\sum_{i=1}^{N_A}t_{\text{isect}}(a_i) + p_B\sum_{i=1}^{N_B}t_{\text{isect}}(b_i)
$
where \\(t_{\text{trav}}\\) is the cost of traversing child nodes and determining which child the ray enters, \\(p_A, p_B\\) are the probabilities of the ray entering A and B, and \\(N_A, N_B\\) are the number of objects in each group.

Our goal is to find a reasonable partition that minimizes \\(c(A, B)\\)—this relies on good estimates of \\(t_{\text{trav}}, t_{\text{sect}}(i)\\), and \\(p\\). In PBRT, the authors make the following assumptions:

- \\(t_{\text{sect}}(i)\\) is a constant and is 8 times \\(t_{\text{trav}}\\).
- \\(p\\) is proportional to the surface area of the bounding box.

In the [PBRT implementation](https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L263-L353), the authors divide the span of the partitioning dimension into 12 segments (resulting in 11 possible partitions), compute \\(c(A,B)\\) for each, and select the one with the lowest expected cost.

## A More GPU-Friendly Approach: HLBVH

The above methods are based on binary partitioning, which is not ideal for highly parallel GPUs. The bottleneck lies in the high cost of determining partitions, which limits us to binary splits. However, if we can assign a deterministic ordering to all objects such that adjacent nodes are spatially close, we can directly sort and split them into any number of segments.

### Mapping High-Dimensional Coordinates to 1D: Morton Code

For high-dimensional integer coordinates (e.g., 3D \\((x, y, z)\\)), we first represent each coordinate in binary form:
$
(\overline{x_nx_{n-1}\cdots x_1x_0}, \overline{y_ny_{n-1}\cdots y_1y_0}, \overline{z_nz_{n-1}\cdots z_1z_0})
$
Then, we interleave the bits to obtain:
$
\cdots z_2y_2x_2z_1y_1x_1z_0y_0x_0
$
Finally, we convert this into an unsigned integer, resulting in the Morton code representation of the coordinate. Morton codes have excellent locality. For example, consider the 2D Morton codes in the following figures:

<div class="side-by-side-container"><img id="should-invert" src="https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/Morton%20Split%20Horiz.svg" style="width: 24%" /><img id="should-invert" src="https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/Morton%20Split%20Vert.svg" style="width: 24%" /></div>
<div class="side-by-side-container"><img id="should-invert" src="https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/Morton%20Shaded%20Region.svg" style="width: 24%" /><img id="should-invert" src="https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/Morton%20Split%20Horiz%203.svg" style="width: 24%" /></div>

- (a) The highest bit being 1/0 corresponds to a horizontal midline partition.
- (b) The second-highest bit being 1/0 corresponds to a vertical midline partition.
- (c) The highest bit being 1 and the second-highest bit being 0 selects the four points in the second quadrant.
- (d) The third-highest bit corresponds to a horizontal quarter partition.

In practice, when computing Morton codes, we first normalize the global bounding box containing all objects and divide it into 1024 segments. We then map the center coordinates of each object’s bounding box to \\([0, 2^{10})\\) and obtain a 30-bit Morton code, which fits into a `uint32_t`.

After obtaining all Morton codes, we perform a radix sort and divide the objects into groups with a fixed stride. This results in non-overlapping groups where objects within each group are spatially close. Building BVHs within each group can be done in parallel across groups.

### Two-Level Tree Construction: HLBVH

First, build trees within each group to obtain many [LBVH Treelets](https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L450-L502). Then, construct a tree from these Treelets using the SAH method, resulting in an [HLBVH](https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L388-L427).

## Compression of BVH

The tree structure of BVH is logically clear but not memory-contiguous, leading to poor memory access performance. Therefore, after building the BVH, we can compress it into a contiguous array in memory, using indices instead of pointers. In this representation, the successor of each node is its left child, and we only need to record the index of the right child (or the objects represented by the node if it’s a leaf).

<div class="side-by-side-container"><img id="should-invert" src="https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/BVH%20linearization.svg" style="width: 75%" /></div>

The compression process is [a simple DFS](https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L504-L521).

## Querying BVH

When querying for intersections with a given ray, we manually maintain a stack. Each time we visit a node, we check if the ray intersects with the node’s bounding box. If not, we exit. If it does, we push its right child onto the stack and visit its left child.

[This algorithm](https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L528-L578) can be implemented using only loops, avoiding the overhead of recursion.
