
**Paper Title:** FEDERAL INFORMATION PROCESSING STANDARDS PUBLICATION 

**Author / Issuing Organization:** National Institute of Standards and Technology (NIST) — Information Technology Laboratory (U.S. Department of Commerce) 

**Publisher / Target:** U.S. Department of Commerce, National Institute of Standards and Technology (NIST)
<<<<<<< HEAD
**Year:** 2015  

-------------------------------------------------------------------------------------------------------------------------------------------------
**1. The Core Functions**
NIST introduced the SHA-3 family not to replace SHA-2, but to act as a solid backup plan. It uses a completely different architecture to stay safe just in case someone finds a major flaw in the older algorithms.

Fixed-Length Hash Functions:

* Includes SHA3-224, SHA3-256, SHA3-384, and SHA3-512.

* They act as easy, drop-in replacements for SHA-2 since the output lengths match up perfectly.

* They provide strong collision and preimage resistance based on their output lengths.

Extendable-Output Functions (XOFs):

* Includes SHAKE128 and SHAKE256.

* These can generate variable-length outputs, which is great for things like key derivation.

Note: The numbers (128 or 256) represent their overall security strength, not how long the output is.

**2. The Sponge Construction**
SHA-3 ditches the old Merkle-Damgård structure (which SHA-1 and SHA-2 used) and uses something called the Sponge Construction. It has a fixed-size internal state and works in two main steps:

* Absorb Phase: It basically "soaks up" the input message by breaking it into chunks (called the Rate) and mixing it into the state using an XOR operation, followed by a mixing function.

* Squeeze Phase: It "wrings out" the output in chunks. If you need a longer output (like with XOFs), it just keeps applying the mixing function and squeezing out more bits.

* The Big Trade-off: The internal state is split into two parts: Rate and Capacity. A higher Rate means faster processing, but a higher Capacity gives you better security.

**3. How KECCAK Works Under the Hood**
The real heavy lifting of the algorithm happens in the KECCAK-p permutations. It treats the data like a 3D block (5x5xW) and scrambles it through multiple rounds using five specific steps:

* Theta: Mixes up the columns to spread the data around (diffusion).

* Rho: Rotates the bits for extra dispersion.

* Pi: Shuffles and rearranges the lanes.

* Chi: The only non-linear step—it substitutes bits along the rows to make the math unpredictable.

* Iota: Adds a round constant so that all the rounds don't look perfectly symmetrical.

**4. Padding and Domain Separation**
Multi-rate Padding (pad10*1): To make sure the message fits perfectly into the required block size, it tacks on a 1, fills the rest with 0s, and ends with another 1.

Domain Separation: To stop cross-protocol attacks (like someone tricking a SHA3-256 hash into matching a SHAKE256 output), it appends a specific suffix right before padding:

* 01 for standard SHA-3 hashes.

* 11 for SHAKE XOFs.

**My Personal Takeaways**
1. Architectural Diversity is a Must
Moving away from the Merkle-Damgård structure shows a golden rule in cryptography: don't put all your eggs in one basket. By standardizing an algorithm with totally different math than SHA-2, NIST made sure that a single hacker breakthrough won't break all of our modern hashing standards at once.

2. Domain Separation Actually Matters
Learning about prefix vulnerabilities in XOFs was a huge eye-opener. If a shorter output is just a prefix of a longer one, attackers could reuse signatures across different protocols. It really drove home why you have to cleanly separate use cases (like using the 01 vs 11 suffixes).

3. Speed vs. Security is Literally Built-in
The relationship between Rate and Capacity in the Sponge construction is such a cool, tangible way to see the performance-vs-security trade-off. If you want a bigger Capacity (more security), you have to shrink the Rate (less speed). You are quite literally spending your system's throughput to buy better security margins.
