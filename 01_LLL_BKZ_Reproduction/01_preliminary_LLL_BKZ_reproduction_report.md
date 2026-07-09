# Reproducing LLL and BKZ for Lattice Reduction

## 1. Goal before the next group meeting

This document records the current plan for reproducing LLL and BKZ/BKZ 2.0 on
simple lattice reduction tasks, and for producing summary tables or figures.

Tasks:

1. Read the related papers and identify how each paper can be used in this
   project.
2. Install WSL and prepare a Linux-based lattice experiment environment.
3. Use LLL and BKZ/BKZ 2.0 to solve simple lattice reduction instances.
4. Plot summary figures such as `||b_1|| / GH(L)` versus dimension.

## 2. Background

A lattice is the set of all integer combinations of basis vectors:

```text
L(B) = { z_1 b_1 + ... + z_n b_n : z_i in Z }.
```

The Shortest Vector Problem (SVP) asks for the shortest non-zero vector in a
lattice. In lattice-based cryptography, exact SVP is usually too hard, so
reduction algorithms such as LLL and BKZ are used to find short vectors or
improve a bad basis.

For experiments, a common quality metric is:

```text
norm/GH = ||b_1|| / GH(L)
```

where `||b_1||` is the length of the first basis vector after reduction, and
`GH(L)` is the Gaussian heuristic estimate of the expected shortest vector
length. Smaller `norm/GH` means better reduction quality. Values closer to 1
are usually better.

## 3. LLL algorithm

LLL is a polynomial-time lattice basis reduction algorithm. It mainly performs:

1. Size reduction: reduce Gram-Schmidt coefficients so basis vectors are not
   unnecessarily correlated.
2. Lovasz condition checks: swap adjacent vectors if their order is bad.

LLL is fast and easy to run, but the quality is weaker than BKZ in higher
dimensions. In this project, LLL can be used as:

- a baseline algorithm;
- a preprocessing step before BKZ;
- a sanity check for lattice instance generation and plotting code.

## 4. BKZ and BKZ 2.0

BKZ can be viewed as a block version of LLL. Instead of only improving adjacent
vectors, BKZ repeatedly considers local blocks of size `beta` and solves or
approximately solves an SVP subproblem inside each block.

Typical experimental settings:

```text
BKZ-20, BKZ-30, BKZ-40
```

The block size controls the time-quality tradeoff:

- larger block size gives better reduction quality;
- larger block size is much slower;
- BKZ 2.0 improves practical performance using techniques such as pruning,
  preprocessing, and better enumeration strategies.

In this project, BKZ can be used to compare how block size changes the quality
of the reduced basis.

## 5. Literature role in this project

| Paper/document | Main role in this project | Possible application |
|---|---|---|
| LLL Algorithm for Lattice Basis Reduction | Understand the baseline reduction algorithm | Implement or call LLL; explain size reduction and Lovasz condition |
| LLL algorithm notes | Supplementary understanding and formula checking | Verify notation, examples, and implementation details |
| Practical Improvements on BKZ | Understand practical BKZ/BKZ 2.0 improvements | Compare BKZ block sizes and explain why larger blocks are more expensive |
| DRL/BKZ related papers | Future extension beyond classical BKZ | Use reinforcement learning to choose BKZ parameters or strategies |

## 6. Reproduction plan

### 6.1 Environment

Recommended WSL Ubuntu setup:

```bash
sudo apt update
sudo apt install -y python3-pip python3-venv git build-essential
python3 -m venv ~/svp-env
source ~/svp-env/bin/activate
pip install numpy matplotlib fpylll cysignals
```

Check installation:

```bash
python - <<'PY'
import numpy, matplotlib, fpylll
print("numpy", numpy.__version__)
print("matplotlib ok")
print("fpylll ok")
PY
```

### 6.2 Lattice instances

Start with simple q-ary lattice instances:

```text
B = [ qI  0 ]
    [ A   I ]
```

where `A` is randomly sampled. These instances are useful because they are
simple, reproducible, and related to cryptographic lattice settings.

### 6.3 Algorithms to run

Run the following algorithms on the same generated instances:

```text
LLL(delta=0.99)
BKZ-20
BKZ-30
BKZ-40
```

For each dimension and random seed, record:

```text
dimension, seed, algorithm, ||b_1||, GH(L), ||b_1||/GH(L), running time
```

### 6.4 Figures and tables

Suggested figures:

1. Scatter plot: dimension vs `||b_1|| / GH(L)`.
2. Line plot: mean `||b_1|| / GH(L)` by dimension.
3. Runtime plot: dimension vs running time.
4. Summary table comparing LLL, BKZ-20, BKZ-30, BKZ-40.

Expected trend:

- LLL is fastest but has weaker reduction quality.
- BKZ-20 improves over LLL.
- BKZ-30 and BKZ-40 should generally give shorter vectors, but they cost more
  time.

## 7. Minimal Python skeleton using fpylll

```python
from fpylll import IntegerMatrix, LLL, BKZ
import math
import numpy as np


def gaussian_heuristic(n, det):
    return math.sqrt(n / (2 * math.pi * math.e)) * math.exp(math.log(det) / n)


def qary_basis(n, q=4093, seed=0):
    rng = np.random.default_rng(seed)
    m = n // 2
    k = n - m
    top = np.hstack([q * np.eye(m, dtype=int), np.zeros((m, k), dtype=int)])
    bottom = np.hstack([rng.integers(-q//2, q//2 + 1, size=(k, m)), np.eye(k, dtype=int)])
    return np.vstack([top, bottom]), q ** m


def to_integer_matrix(B):
    M = IntegerMatrix(B.shape[0], B.shape[1])
    for i in range(B.shape[0]):
        for j in range(B.shape[1]):
            M[i, j] = int(B[i, j])
    return M


def first_norm(M):
    return math.sqrt(sum(int(M[0, j]) ** 2 for j in range(M.ncols)))


n = 60
B, det = qary_basis(n, seed=0)
gh = gaussian_heuristic(n, det)

M = to_integer_matrix(B)
LLL.reduction(M, delta=0.99)
print("LLL norm/GH:", first_norm(M) / gh)

for beta in [20, 30, 40]:
    N = IntegerMatrix.from_matrix(M)
    BKZ.reduction(N, BKZ.Param(block_size=beta))
    print(f"BKZ-{beta} norm/GH:", first_norm(N) / gh)
```

## 8. Current status

Current local status:

- WSL still needs to be confirmed with an installed Linux distribution.
- `fpylll` is the preferred package for faithful LLL/BKZ reproduction.
- If `fpylll` cannot be installed immediately, a pure Python low-dimensional
  teaching version can still be used to understand the workflow and plotting.

## 9. Next actions

1. Confirm WSL Ubuntu can run `python3`.
2. Install `fpylll`, `cysignals`, `numpy`, and `matplotlib`.
3. Run LLL/BKZ experiments on dimensions 50 to 120.
4. Generate plots and add them to this document.
5. Push this document and the experiment scripts to GitHub.
