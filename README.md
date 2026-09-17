# cluster-nv

Clustering divides a set of observations into groups, without being
told in advance which observation belongs where. This package brings
three of the standard methods to novo-lang: k-means, DBSCAN and
agglomerative clustering. They are the same three
[scikit-learn.cluster](https://scikit-learn.org/stable/modules/clustering.html)
and [linfa-clustering](https://docs.rs/linfa-clustering/) are built
around. The data is an [ndarray-nv](https://novo-lang.org/packages/ndarray-nv)
matrix.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release. `ndarray-nv` 0.0.1, which this package
is built on, is an interface too, so nothing here can be implemented
before it is.

## What it is

The input is a **matrix** of `n_samples` rows and `n_features`
columns: one row per observation, one column per measured quantity. A
**cluster** is a set of rows the method decided belong together, and a
**label** is the cluster number a row was given.

**k-means** is told how many clusters to find, a number written `k`.
It chooses `k` starting points called **centroids**, assigns every
row to the nearest one, moves each centroid to the mean of the rows
assigned to it, and repeats until nothing moves. The quantity it
reduces at every step is the **inertia** — the sum, over every row, of
the squared distance from that row to its own centroid. Where the
centroids start changes the answer, which is why the starting choice
has a name of its own: **k-means++** picks them one at a time, each
with a probability proportional to its squared distance from the
nearest centroid already chosen.

**DBSCAN** — Density-Based Spatial Clustering of Applications with
Noise, published by Ester, Kriegel, Sander and Xu in 1996 — is not
told how many clusters to find. It is told a radius, called
**epsilon**, and a count, called **min_points**. A row with at least
`min_points` rows within `epsilon` of it, itself included, is a **core
point**. Core points within `epsilon` of each other are in the same
cluster, and a non-core row within `epsilon` of a core point joins its
cluster as a **border point**. Every other row is **noise**: it is in
no cluster, and it is not a cluster of its own.

**Agglomerative clustering** starts with every row its own cluster and
repeatedly merges the two closest, recording each merge and the
distance it happened at. That record is a **dendrogram**, a binary
tree whose height at each node is the distance between what it joined.
Cutting the tree gives a clustering, and the tree can be cut at any
height without being rebuilt. "The distance between two clusters"
needs a definition, called the **linkage**: the smallest distance
between any pair of their points (**single**), the largest
(**complete**), or the mean over all pairs (**average**).

## Install

```
novo pkg add cluster-nv
```

## Example

```novo
use clusterkmeans
use clusterlabels
use ndfloat

fn main() [io]
    // Four measurements: two near zero and two near ten.
    match ndfloat.of_list([0.0, 1.0, 10.0, 11.0], [4, 1])
        Err(e) => println(e.message())
        Ok(data) =>
            // The random draws k-means++ needs, one per centroid. A
            // caller reads them from its own generator; here they are
            // written down, so this program prints the same thing
            // every time it runs.
            let draws = [0.0, 0.9]

            // Seed and fit in one call: at most 100 passes, and stop
            // when no centroid moves further than a millionth.
            match clusterkmeans.fit_seeded(data, 2, draws, 100, 0.000001)
                Err(e) => println(e.message())
                Ok(model) =>
                    // The status is read before the labels: labels
                    // from an exhausted budget are not a fixed point.
                    println(clusterlabels.status_name(model.status))
                    println("${clusterlabels.labels_of(model.labels)}")   // [0, 0, 1, 1]
                    println("inertia ${model.inertia}")                   // 1
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: cluster-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `clusterfault` | Every way a call is refused before the first distance is computed, and the numbers that refused it. |
| `clusterlabels` | The labels a clustering answers, the noise label, the four distance metrics and the three-armed status. |
| `clusterkmeans` | k-means, k-means++ seeding from caller-supplied draws, and the assign, mean and inertia steps on their own. |
| `clusterdbscan` | DBSCAN, its two parameters as a value, the neighbourhood query and the k-distance plot. |
| `clusteragglomerative` | The merge tree, the three linkages, and the two ways to cut a tree. |

## How to choose an entry point

**You know how many clusters there are: `clusterkmeans.fit_seeded`.**
It is the fastest of the three and it assigns every row. It assumes the
clusters are roughly round and roughly the same size, and it cuts
elongated or nested shapes in half.

**You know how dense a cluster is, but not how many there are:
`clusterdbscan.fit`.** It follows whatever shape is dense, finds as
many clusters as the data supports, and leaves rows in sparse regions
unassigned. Use `clusterdbscan.k_distances` first to choose `epsilon`:
plotted from largest to smallest, it has a knee, and the distance at
the knee is the radius the data is asking for.

**You want to see the structure before choosing:
`clusteragglomerative.build`, then cut.** Building the tree is the
expensive part and it is done once; `cut_at_count` and
`cut_at_distance` are cheap and can be called as often as you like on
the same tree. Use `merge_heights` to find where the natural division
is — the largest gap between consecutive heights.

**You already have labels and want to score them:
`clusterkmeans.centroids_of` then `clusterkmeans.inertia`.** Both work
on any labelling, wherever it came from.

## The rules a user needs

1. **The random draws are yours.** `clusterkmeans.seed_plus_plus`
   takes a list of uniform numbers in `[0, 1)`, and
   `clusterkmeans.uniforms_needed(k)` says how many. Draw them from
   whatever generator you already have. Two runs with the same data
   and the same draws give the same clustering, on every machine, with
   no seed to set.
2. **Read the status before the labels.** `ClusterBudgetExhausted`
   means the iteration stopped early and another pass would still move
   rows. `ClusterDegenerate` means it produced fewer clusters than
   asked for, and says why.
3. **Noise is not a cluster.** Use `clusterlabels.is_noise` rather than
   comparing a label against a number, and
   `clusterlabels.cluster_count` rather than taking the maximum label
   plus one. `clusterlabels.sizes` has one entry per cluster and none
   for the noise.
4. **k-means takes no metric.** Its update step is the mean, and the
   mean minimises the squared Euclidean distance and nothing else.
   Under the Manhattan distance the minimiser is the median and the
   algorithm is a different one. DBSCAN and agglomerative clustering
   accept all four metrics, because neither of them ever takes a mean.
5. **`epsilon` is in the units of your data.** A radius that is too
   small labels everything noise; one that is too large makes
   everything one cluster. `clusterlabels.noise_count` is the first
   number to look at after a fit, and it says which mistake was made.
6. **The linkage is a statement about shape, not a tuning knob.**
   Single linkage follows filaments and will join two dense groups
   through one bridge of outliers. Complete linkage produces compact
   clusters of similar diameter and breaks elongated ones up. The
   three give genuinely different trees on the same data.
7. **Ties are broken by index, and the data's row order decides.**
   Two equally close pairs of clusters merge lower-index-first; DBSCAN
   numbers its clusters in the order it discovers them, scanning rows
   in order; a border point reachable from two clusters joins the one
   that reached it first. All three are stated so that the same data
   always gives the same answer.
8. **The data must be rank 2.** A rank-1 array of measurements is
   refused rather than guessed at, because only the caller knows
   whether it is one feature of many samples or many features of one.

## What is not included

- **A random number generator.** Every function that needs randomness
  takes the numbers as an argument. That is what lets this package
  declare no effects at all, and it is what makes every run
  reproducible.
- **Gaussian mixture models, spectral clustering, OPTICS, HDBSCAN,
  mean shift and affinity propagation.** Each is a package's worth of
  work in its own right and none of them changes a signature here.
- **Ward linkage.** It minimises the increase in within-cluster
  variance rather than a distance between points, so it needs the
  Lance–Williams update rather than the pairwise-distance definition
  the other three share. It is the next linkage to add.
- **Cluster quality scores** — silhouette, Calinski–Harabasz,
  Davies–Bouldin, adjusted Rand index. They score a clustering rather
  than produce one, and they belong beside the other statistics.
  `clusterlabels.distance` and `clusterkmeans.inertia` are public so
  that a caller can compute them without reimplementing the
  arithmetic the fit used.
- **A spatial index.** Every neighbourhood query here compares every
  pair, which is right for thousands of rows and wrong for millions.
  An R-tree or a k-d tree underneath would change the running time and
  no signature.
- **Anything that reads or writes.** No progress callback, no
  iteration trace, no file. All three would be `[io]`, and this
  package's whole surface is `[]`.

## Related packages

- **`ndarray-nv`** is the matrix this package's data is. Convert at
  the boundary and the two compose with no copy.
- **`optimize-nv`** minimises a caller-supplied function. k-means is a
  minimisation too, of the inertia, but a specialised one whose two
  steps each reduce it exactly — a general minimiser would be slower
  and would need a gradient the problem does not have.
- **`stats-nv`** summarises and tests. A clustering is often the step
  before a per-cluster summary.

## Tests

The suite is written against the signatures and is red by
construction: every assertion reaches a `todo()`. Its fixtures are
small enough to work out by hand and are chosen so that every asserted
number is arithmetic rather than a value copied from a run. Four
points on a line — 0, 1, 10, 11 — are two well-separated pairs whose
k-means fit has centroids at 0.5 and 10.5 and an inertia of 1, and
whose dendrogram's last merge is at 9, 10 or 11 depending on the
linkage. Five points — three tight, two apart, one far away — give
DBSCAN two clusters and exactly one noise point. The seeding draws are
written into the tests, which is what taking them as an argument buys:
there is no generator to fake and no global to reset.

## Implementation status

| Module | Declared | Implemented |
| --- | --- | --- |
| `clusterfault` | 13 variants, `message` | no |
| `clusterlabels` | 13 functions | no |
| `clusterkmeans` | 8 functions | no |
| `clusterdbscan` | 6 functions | no |
| `clusteragglomerative` | 7 functions | no |

## Licence

Apache-2.0. See [LICENSE](LICENSE).
