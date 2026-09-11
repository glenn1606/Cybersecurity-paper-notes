
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
