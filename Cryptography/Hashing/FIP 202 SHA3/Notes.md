
**Paper Title:** FEDERAL INFORMATION PROCESSING STANDARDS PUBLICATION 

**Author / Issuing Organization:** National Institute of Standards and Technology (NIST) — Information Technology Laboratory (U.S. Department of Commerce) 

**Publisher / Target:** U.S. Department of Commerce, National Institute of Standards and Technology (NIST)
<<<<<<< HEAD
**Year:** 2015  

-------------------------------------------------------------------------------------------------------------------------------------------------


**Summary (Introduction & Glossary):**

* **Purpose:** Standardizes the SHA-3 family of functions based on the KECCAK algorithm (winner of the NIST SHA-3 Cryptographic Hash Algorithm Competition) to complement existing SHA-1 and SHA-2 standards.

* **Standardized Functions:**

   4 Cryptographic Hash Functions: SHA3-224, SHA3-256, SHA3-384, and SHA3-512 with fixed output digest lengths.

   2 Extendable-Output Functions (XOFs): SHAKE128 and SHAKE256, allowing arbitrary output lengths tailored to application requirements.

* **Core Architecture:** Built upon the sponge construction using underlying KECCAK-p mathematical permutations.
# SHA-3 / KECCAK Permutation Specification

> Tài liệu tóm tắt & chuẩn hóa kỹ thuật hoán vị KECCAK-p (Theo chuẩn FIPS 202)

---

# KECCAK-p / SHA-3 Permutation Summary

## 1. Core Parameters & Terms
* **State ($A$)**: $5 \times 5 \times w$ array, where $w = b/25 \in \{1, 2, 4, 8, 16, 32, 64\}$ bits (*SHA-3 default: $b=1600, w=64$*).
* **Capacity ($c$)**: $c = b - r$ ($b$: width, $r$: rate).
* **Key Subparts**:
  * **Lane**: $w$-bit word at $(x, y)$. Maps to a single 64-bit CPU word when $b=1600$.
  * **Slice**: 25-bit array at fixed $z$. Center at $(0, 0)$.

## 2. Permutation Mechanics
* **Round Function ($\text{Rnd}$)**: 5 sequential step mappings:
  $$\theta \longrightarrow \rho \longrightarrow \pi \longrightarrow \chi \longrightarrow \iota$$
  * $\theta, \rho, \pi, \chi$: Round-independent linear/non-linear transformations.
  * $\iota$: Injects round constants (round-dependent).

* **State Indexing**: $1\text{D Bitstring } (S) \leftrightarrow 3\text{D Array } (A)$
  $$A[x, y, z] = S[w(5y + x) + z]$$
  * Reconstructed via concatenation: $\text{Lanes} \to \text{Planes} \to \text{Full State } (S)$.

# Permutations KECCAK-p (Section 3)

## 1. Core State Structure (Section 3.1)
- **State Array (A):** 3D array of size 5 x 5 x w, where w = 2^l is the lane length (w in {1, 2, 4, 8, 16, 32, 64}).
- **Total State Width (b):** b = 5 x 5 x w = 25w bits (standard SHA-3 uses b = 1600, w = 64).
- **Key Concepts:**
  - `Lane`: A[i, j, :] (1D array of w bits along z-axis)
  - `Row`: A[:, j, z] (5 bits along x-axis)
  - `Column`: A[i, :, z] (5 bits along y-axis)
  - `Slice`: A[:, :, z] (5x5 matrix of 25 bits at a fixed z)

---

## 2. Step Mappings / Round Transformation (Section 3.2)
Each round applies 5 step mappings sequentially: Rnd(A, ir) = iota(chi(pi(rho(theta(A)))), ir).

### 2.1 Theta - Linear Diffusion
- **Purpose:** Linear diffusion across columns.
- **Mechanism:** XOR each bit with the parity of two adjacent columns (x-1 and x+1).
- **Operation:**
  1. C[x, z] = A[x, 0, z] XOR A[x, 1, z] XOR A[x, 2, z] XOR A[x, 3, z] XOR A[x, 4, z]
  2. D[x, z] = C[(x-1) mod 5, z] XOR C[(x+1) mod 5, (z-1) mod w]
  3. A'[x, y, z] = A[x, y, z] XOR D[x, z]

### 2.2 Rho - Intra-Lane Bit Rotation
- **Purpose:** Bit dispersion along the z-axis (time/position dispersion).
- **Mechanism:** Rotates bits within each lane by a fixed offset.
- **Operation:**
  - A'[x, y, z] = A[x, y, (z - offset[x, y]) mod w]
  - Offset at (0, 0) is 0; other 24 offsets are predetermined constants.

### 2.3 Pi - Inter-Lane Permutation
- **Purpose:** Spatial mixing across x and y coordinates.
- **Mechanism:** Permutes lane positions in the 5 x 5 grid.
- **Operation:**
  - A'[(x + 3y) mod 5, x, z] = A[x, y, z]
  - Keeps (0, 0) fixed.

### 2.4 Chi - Non-Linear Layer
- **Purpose:** Provides cryptographic non-linearity (S-box equivalent).
- **Mechanism:** Operates row-wise (x-axis) using AND, NOT, and XOR.
- **Operation:**
  - A'[x, y, z] = A[x, y, z] XOR ((NOT A[(x+1) mod 5, y, z]) AND A[(x+2) mod 5, y, z])

### 2.5 Iota - Symmetry Breaking
- **Purpose:** Destroys structural symmetries across rounds.
- **Mechanism:** XORs a round constant RC[ir] into the origin lane A[0, 0].
- **Operation:**
  - A'[0, 0, z] = A[0, 0, z] XOR RC[ir][z]
  - All other 24 lanes remain unchanged.

---

## 3. Permutation Construction & Parameters (Section 3.3 - 3.4)
- **KECCAK-p[b, nr]:** Parameterized by width b and number of rounds nr.
- **KECCAK-f[b]:** Special case of KECCAK-p where nr = 12 + 2l.
- **SHA-3 Instance:** Uses KECCAK-p[1600, 24] (equivalent to KECCAK-f[1600]), where w = 64, l = 6, nr = 12 + 2(6) = 24.