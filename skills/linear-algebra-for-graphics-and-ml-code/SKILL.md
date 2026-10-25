---
name: linear-algebra-for-graphics-and-ml-code
description: Apply vectors, matrices, linear transformations, linear-system solving, and eigenvalues/eigenvectors to real code — graphics pipelines, ML feature engineering (PCA), physics/game math, and "why is my matrix singular" debugging. Use this whenever the user asks how to rotate/scale/translate points, wants to combine multiple transforms into one matrix, needs to solve a system of linear equations (Ax=b), asks "what's this matrix's rank/determinant/eigenvectors for," wants to vectorize per-point math with numpy instead of Python loops, is implementing PCA or dimensionality reduction from scratch, or is debugging a "singular matrix" / "no solution" / numerically unstable linear-algebra bug. Also triggers on "dot product vs cross product," "homogeneous coordinates," "why use np.linalg.solve instead of inv()," and "eigenvector meaning."
---

# Linear Algebra for Graphics and ML Code

Linear algebra is the language for "many numbers, transformed together, all at once." The
moment you're tracking more than one coordinate per object (position, color, feature vector,
weight matrix) and applying the same operation to all of them, you're doing linear algebra —
whether or not you reach for `numpy`. This skill covers the operations that actually show up in
application code: vectors and their products, matrices as transformations, solving linear
systems without triggering the classic numerical-instability bugs, and eigenvalues/eigenvectors
for the handful of places they matter in practice (PCA, stability analysis, iterated systems).

## Vectors: the basic unit

A vector is an ordered list of numbers with two operations that matter: addition (combine) and
scalar multiplication (scale). Everything else — dot product, cross product, normalization — is
built from those.

```python
import numpy as np

v = np.array([3.0, 4.0])
w = np.array([1.0, 0.0])

magnitude = np.linalg.norm(v)          # 5.0  — sqrt(3^2 + 4^2)
unit_v = v / magnitude                  # normalize: direction with length 1
dot = np.dot(v, w)                      # 3.0  — projection-related scalar
```

**Dot product** answers "how much do these two vectors point the same way?" `a·b = |a||b|cos θ`.
Use it for: projecting one vector onto another, computing the angle between vectors, lighting
calculations (`dot(normal, light_dir)` for diffuse shading), and similarity between feature
vectors (cosine similarity is just a normalized dot product).

**Cross product** (3D only) answers "what's perpendicular to both of these?" `a × b` is a vector
orthogonal to both `a` and `b`, with magnitude `|a||b|sin θ`. Use it for: surface normals from two
edge vectors, torque/angular quantities, and testing which side of a line/plane a point is on
(sign of the cross product).

```python
a = np.array([1.0, 0.0, 0.0])
b = np.array([0.0, 1.0, 0.0])
normal = np.cross(a, b)   # [0, 0, 1] — perpendicular to the a-b plane
```

**The mixing-them-up bug**: reaching for dot product when you need "perpendicular to both" (it
doesn't give you that — it gives you a scalar) or reaching for cross product in 2D code (there's
no 3D cross product in 2D; the 2D analog, `ax*by - ay*bx`, gives a scalar "signed area," not a
vector).

## Matrices as transformations, not just number grids

The mental model that pays off: a matrix `M` is a function that takes a vector `v` and produces
another vector `Mv`. Rotation, scaling, shearing, and projection are all just particular
matrices. This is why you can build a complex transform (rotate, then scale, then translate) by
**multiplying matrices together** and applying the combined result once, instead of applying each
step separately to every point.

```python
import numpy as np

def rotation_2d(theta):
    c, s = np.cos(theta), np.sin(theta)
    return np.array([[c, -s],
                      [s,  c]])

def scale_2d(sx, sy):
    return np.array([[sx, 0],
                      [0, sy]])

# Combine: scale first, then rotate. Matrix multiplication order matters —
# transforms apply right-to-left when points are column vectors.
combined = rotation_2d(np.pi / 4) @ scale_2d(2.0, 1.0)
point = np.array([1.0, 0.0])
transformed = combined @ point
```

**Order is not commutative.** `rotate @ scale` and `scale @ rotate` produce different results —
rotating a stretched square is not the same as stretching a rotated square. When a transform
pipeline looks right in isolation but wrong when combined, check the multiplication order first.

**Translation needs homogeneous coordinates.** A pure `2x2` (or `3x3` in 3D) matrix can rotate and
scale but cannot translate — there's no way to add a constant offset via multiplication alone.
The standard trick: append a `1` to every point and use one extra row/column so translation
becomes part of the matrix multiply:

```python
def translate_2d(tx, ty):
    return np.array([[1, 0, tx],
                      [0, 1, ty],
                      [0, 0, 1]])

def to_homogeneous(point):
    return np.array([point[0], point[1], 1.0])

# Now rotation, scaling, AND translation all compose via one matrix product.
M = translate_2d(5, 0) @ rotation_2d(np.pi / 2)
p = to_homogeneous(np.array([1.0, 0.0]))
result = (M @ p)[:2]   # drop the homogeneous 1 at the end
```

If translation "isn't working" in a transform pipeline that otherwise looks correct, the missing
homogeneous coordinate is the first thing to check.

## Solving linear systems: use `solve`, not `inv`

`Ax = b` shows up constantly: intersecting lines/planes, least-squares fitting, balancing a
system of constraints. The tempting approach — compute `A⁻¹` then `x = A⁻¹b` — is both slower and
less numerically stable than solving directly.

```python
import numpy as np

A = np.array([[3, 2], [1, -1]])
b = np.array([12, 1])

x = np.linalg.solve(A, b)          # preferred: numerically stable, O(n^3) once
# x_bad = np.linalg.inv(A) @ b     # avoid: computes a full inverse you don't need,
                                    # amplifies rounding error, fails ungracefully
```

`np.linalg.solve` uses Gaussian/Gauss-Jordan-style elimination under the hood without forming
the explicit inverse, which is both faster and better-conditioned. Reach for the explicit
inverse only when you genuinely need the inverse matrix itself (e.g., for a formula that reuses
it many times) — not as a step toward solving one system.

**Singular and near-singular matrices**: a system has no unique solution when `A`'s rows/columns
are linearly dependent — geometrically, the transformation collapses space into a lower dimension
(a plane onto a line, etc.), so information is lost and can't be inverted. Symptoms in code:
`np.linalg.solve` raises `LinAlgError: Singular matrix`, or (worse) silently returns a wildly
wrong answer because the matrix is *nearly* singular (small determinant, large condition number)
rather than exactly singular.

```python
# Never test invertibility with an exact float comparison:
if np.linalg.det(A) == 0:      # WRONG — floating point determinants are rarely exactly 0
    ...

# Better: check the condition number, or check rank against expected dimension.
cond = np.linalg.cond(A)        # large (>~1e10) means "numerically singular for practical purposes"
rank = np.linalg.matrix_rank(A) # rank < A.shape[0] means genuinely singular
```

A matrix's **rank** is the number of linearly independent rows/columns — it tells you the
dimension of the space the transformation actually maps onto. Rank-deficient data (e.g., a
feature matrix where one column is a linear combination of others) is why some ML models
silently drop or ignore redundant features and why some linear-regression fits are unstable
until the redundant column is removed.

## Eigenvalues and eigenvectors: what they're actually for

An eigenvector of `M` is a direction that `M` doesn't rotate — it only scales it, by the
corresponding eigenvalue: `Mv = λv`. Most vectors get rotated *and* scaled by a transform;
eigenvectors are the special directions where only scaling happens. This shows up in three
concrete places:

1. **PCA / dimensionality reduction** — the eigenvectors of a dataset's covariance matrix are the
   directions of maximum variance ("principal components"); the corresponding eigenvalues tell you
   how much variance each direction explains.
2. **Repeated/iterated transforms** — applying the same matrix over and over (Markov chain
   transitions, iterated affine systems, PageRank-style graph centrality) converges toward the
   eigenvector with the largest eigenvalue — that's *why* stationary distributions and dominant
   modes exist.
3. **Stability analysis** — for a linear system `x' = Ax`, the eigenvalues of `A` determine whether
   the system converges to equilibrium, oscillates, or blows up.

```python
import numpy as np

# PCA by hand: center the data, get the covariance matrix, eigen-decompose it.
X = np.array([[2.5, 2.4], [0.5, 0.7], [2.2, 2.9], [1.9, 2.2], [3.1, 3.0]])
X_centered = X - X.mean(axis=0)
cov = np.cov(X_centered, rowvar=False)

eigenvalues, eigenvectors = np.linalg.eig(cov)

order = np.argsort(eigenvalues)[::-1]           # largest eigenvalue first
eigenvalues, eigenvectors = eigenvalues[order], eigenvectors[:, order]

principal_component = eigenvectors[:, 0]         # direction of max variance
projected = X_centered @ principal_component     # 1D projection onto it
```

**Eigenvector sign/scale is not unique.** `np.linalg.eig` might return `v` or `-v` for the same
eigenvector, and any nonzero scalar multiple is equally valid mathematically — code that compares
eigenvectors for equality (rather than for being parallel) will intermittently fail across numpy
versions or platforms.

## Common pitfalls

- **Computing `inv(A) @ b` instead of `solve(A, b)`** — slower and less numerically stable; use
  `solve` unless you need the inverse matrix itself for reuse.
- **Testing singularity with `det(A) == 0`** — floating-point determinants are essentially never
  exactly zero; check `matrix_rank` or `cond` instead.
- **Forgetting homogeneous coordinates** and wondering why translation "doesn't compose" with
  rotation/scaling in a matrix pipeline.
- **Assuming matrix multiplication commutes** — `rotate @ scale != scale @ rotate`; when a
  combined transform looks wrong, check the order first.
- **Confusing dot and cross product** — dot gives a scalar ("how aligned"), cross (3D only) gives
  a perpendicular vector ("how twisted").
- **Comparing eigenvectors for exact equality** across implementations/runs — compare direction
  (e.g., via dot product magnitude near 1) instead, since sign and scale are arbitrary.
- **Ignoring rank deficiency in feature data** — a linear model fit on collinear features is
  numerically unstable even when it "runs without error."

## Checklist

- [ ] Systems `Ax = b` are solved with `np.linalg.solve` (or an appropriate `lstsq` for
      overdetermined/least-squares systems), not via an explicit matrix inverse
- [ ] Singularity/invertibility checks use rank or condition number, never an exact `== 0`
      float comparison
- [ ] Transform pipelines that need translation use homogeneous coordinates, and the
      multiplication order matches the intended apply-order (rightmost matrix applies first to a
      column vector)
- [ ] Eigenvector-based code compares directions (parallelism), not exact vector equality
- [ ] Feature matrices for ML are checked for rank deficiency (redundant/collinear columns)
      before fitting a linear model
- [ ] Dot product and cross product are not swapped — dot for "how aligned," cross for
      "perpendicular to both" (3D only)
