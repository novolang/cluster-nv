# Changelog

All notable changes to cluster-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-10

**The implementation.**  Every body under `src/` is written, and every
signature, effect row and doc comment is the one the interface release
published.  The surface is still `[]`: nothing here reads a clock,
opens a file or prints a trace, and the draws k-means++ needs are
still an argument rather than an effect.

The algorithms are the ones the interface describes.  What the tests
pin, and what a reader should know before trusting a number, is the
part the signatures could not state: which side of a tie wins, how
clusters are numbered, and what a method answers when it cannot finish.

### Added

- `clusterkmeans` — the four steps written out: assignment to the
  nearest centroid under the squared Euclidean distance, the mean of
  each assignment, the inertia, and the iteration that alternates them
  until no centroid moves further than the tolerance.  Seeding is
  k-means++ over the caller's draws, one per centroid: the first
  picks a row uniformly, and each later one picks with a weight
  proportional to its squared distance from the nearest centroid
  already chosen.  The iteration ends `ClusterConverged`,
  `ClusterBudgetExhausted`, or `ClusterDegenerate` when an assignment
  leaves a cluster with no members.  `assign`, `cluster_means`,
  `inertia`, `predict` and `centroids_of` are the steps on their own,
  so a caller can score a labelling that came from somewhere else.
- `clusterdbscan` — one pass over the rows in order.  A row with
  `min_points` neighbours inside `epsilon` is a core point, a
  non-core row inside a core point's neighbourhood is a border point
  of that cluster, and every other row is noise, answered as
  `noise_label()` and counted apart from the clusters.  The pass
  yields its answer at once, so the status is always
  `ClusterConverged`.  `neighbours`, `is_core_point` and
  `k_distances` are the queries the fit is built from, and all four
  metrics are accepted here because DBSCAN never takes a mean.
- `clusteragglomerative` — every pair's distance under the chosen
  metric, then the closest two clusters merged repeatedly with the
  Lance-Williams update for single, complete and average linkage.
  Each merge records the two clusters it joined, smaller number
  first, and the distance it happened at; that recording is what makes
  equally close pairs merge lower-index-first and the tree
  deterministic.  A cut replays the merge records, so a tree is built
  once and cut as often as a caller likes, and the labels are numbered
  from 0 by the first sample in each cluster rather than by the merge
  numbers.
- `clusterlabels` — `distance` for all four metrics, the noise label
  and its predicate, `sizes` with one entry per cluster and none for
  the noise, and `status_name` and `converged` over the three arms of
  `ClusterStatus`.
- `clusterfault` — all thirteen messages, each naming the numbers that
  refused the call, with ndarray-nv's own fault carried unchanged.
- `tests/cluster_edge_tests.nv` — the second suite: the shape guards
  every entry point opens with, the arguments that are refused, the
  metrics and the queries the small fixtures never reach, and one
  merge history where the cluster on the right of a union holds the
  lower root.  `tests/coverage.sh` runs all six suites and prints the
  union of their line coverage over `src/`.

### Known

- **`tests/clusterdbscan_tests.nv` contradicts itself, and no fixture
  satisfies it.**  Its `of_list` call carries five points — 0, 0.1,
  0.2, 10, 50 — while its header describes six, with a 10.1 beside the
  10.  Against the fixture as written two cases fail: line 62 wants two
  clusters and the fit gives one, and line 142 wants a largest
  nearest-neighbour distance of 39.9 and the measurement is 40.0
  (`|40 - 39.9| > 1e-07`).  Adding the 10.1 does not settle it — the
  file then fails four *different* cases, at lines 71, 100, 117 and
  132, because those index position 4 as "the point at 50" and a sixth
  point moves it to 10.1.  So the file needs both layouts at once, and
  the assertions in it are left exactly as written; this entry is the
  report.  The two failures are why `novo pkg publish` reports
  `shard 4/5` on the `test` row.
- **Five lines under `src/` are excused from line coverage**, each by
  a `cov: skip` marker directly above it: `picked_row`'s clamp, the
  `of_list` arm of `seed_plus_plus`, the `of_list` arm and the inner
  `inertia` arm of `fit`, and the `of_list` arm of `centroids_of`.
  Every one is a fallback the module's own construction makes
  unreachable, since each `of_list` is handed exactly the shape it is
  asked for and the clamp is over a draw below 1.0.  The other 520 of
  the 525 instrumented lines run.
- **A doc example in `clusterkmeans` prints a fault rather than the
  number it shows.**  `ClusterKmeansModel`'s example builds its data
  with `ndfloat.of_list([0.0, 0.0, 10.0, 10.0], [2, 1])`, and four
  values under a shape of two elements is `NdSizeMismatch`, so the
  fault arm prints and the `// 2` beside the call never does.  The
  shape should be `[4, 1]`.  The doc comment is the interface, so the
  example is unchanged here.
- **`clusterfault`'s header says "the eleven ways it does"** where the
  enum has thirteen variants.  Left as written for the same reason.
- **`ClusterBadCut` answers for two refusals its own doc does not
  scope it to.**  `centroids_of` and `inertia` use it for a labelling
  of the wrong length, and `centroids_of` for a labelling with an
  empty cluster — the second because the doc of `centroids_of` names a
  `ClusterStatus` arm for that case, and a status is not a fault.

This release requires novo 0.23.0 or later.

## 0.0.3 — 2026-10-25

A file that names a sibling module's type, function or constant now has the `use` line for that module (E2033, E2003).  No signature and no behaviour changed.

## 0.0.2 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- ndarray-nv: `^0.0.1` to `^0.1.4`.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `clusterkmeans` — the load-bearing interface. k-means++ draws its
  starting centroids at random, and `[rand]` is a host effect, so the
  draws are an ARGUMENT: `seed_plus_plus` takes a list of uniforms in
  `[0, 1)` and `uniforms_needed` says how many. That is what keeps the
  row `core`, and it buys three things the usual design cannot — a run
  is reproducible with no seed-setting ritual, a test writes the draws
  it wants and asserts the exact clustering they produce, and the
  caller chooses its own generator. A caller with its own starting
  points skips the seeding and hands `fit` the centroids directly.
  No function here takes a metric: the update step is the mean, and the
  mean minimises the squared Euclidean distance and nothing else.
- `clusterlabels` — the noise label is a NAMED thing. Scikit-learn
  writes it as -1, and the consequence is the well-known bug where a
  program counts clusters by taking the maximum label plus one, or
  indexes a per-cluster array with the label. Here `noise_label`,
  `is_noise`, `cluster_count` and `sizes` between them make that
  unwritable. `ClusterStatus` has the same three arms an iterative
  method always has, so a caller can tell a fixed point from a budget
  that ran out.
- `clusterdbscan` — the two parameters as one value checked where the
  configuration is read, plus `neighbours`, `is_core_point` and the
  k-distance plot, which is how `epsilon` gets chosen rather than
  guessed. Border points are assigned in the data's row order, and the
  module says so, so the original algorithm's visit-order dependence
  becomes a stated rule instead of a surprise.
- `clusteragglomerative` — `build` is separate from the cuts, because
  the tree is the expensive part and the whole value of the method is
  cutting it many times. `cut_at_count` for when the number of
  clusters is known, `cut_at_distance` for when a meaningful distance
  is. Ties merge lower-index-first, so the tree is deterministic.
- `clusterfault` — thirteen refusals, every one about the CALL and
  every one findable before the first distance is computed.
  `ClusterArrayFault` carries ndarray-nv's own `NdFault` unchanged
  rather than rewriting it, so the shapes it compared survive into the
  message and a caller can still match on the original.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  cluster-nv.<module>.<fn>`.
- **`ndarray-nv` 0.0.1 is an interface too.** Every one of its bodies
  is a `todo()`, so nothing here can be implemented before it is. The
  README's Status section names that rather than leaving it to be
  found.
- **No spatial index.** Every neighbourhood query compares every pair,
  which is right for thousands of rows and wrong for millions. An
  R-tree underneath would change the running time and no signature.
- **No Ward linkage, no Gaussian mixtures, no cluster quality
  scores.** Each is named in the README with the reason it is out.
