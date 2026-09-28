# Changelog

All notable changes to cluster-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
