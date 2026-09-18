**Paper Title:** The Art, Science, and Engineering of Fuzzing: A Survey

**Authors:** Valentin J.M. Manès, Hyung Seok Han, Choongwoo Han, Sang Kil Cha, Manuel Egele, Edward J. Schwartz, Maverick Woo

**Year:** 2019 (Submitted/Revised 2019, IEEE TSE 2021)

**Publisher / Venue:** IEEE Transactions on Software Engineering (IEEE TSE) / IEEE

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### **1. Introduction & Context **
* **Overview**: Fuzzing is a popular automated software testing technique that repeatedly runs a program under test (PUT) using syntactically or semantically malformed inputs.
* **Adoption**: Deployed extensively across attackers (exploit generation), defenders, security auditors, and major software vendors (Google, Microsoft, Cisco, Adobe).
* **Motivation**: Rapid growth led to terminology fragmentation (e.g., conflicting usages of "minimization") and undocumented design choices. The authors systematize the literature, propose a unified model fuzzer, and provide a comprehensive taxonomy.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### **2. Core Definitions & Terminology **
* **Fuzzing**: Executing a PUT with inputs sampled from a space that protrudes its expected input space.
* **Fuzz Testing**: Using fuzzing to evaluate whether a PUT violates a specified correctness policy.
* **Fuzzer / Fuzz Campaign**: A program executing fuzz testing / A specific run of a fuzzer on a PUT under a defined policy.
* **Bug Oracle**: A program or mechanism that determines if a test execution violates the correctness policy.
* **Fuzz Configuration**: Parameter values controlling the fuzz algorithm (e.g., seed pool, mutation ratio).

---

### **3. Paper Selection Criteria **
* Surveyed papers published between January 2008 and February 2019 across 4 major security conferences (CCS, S&P, NDSS, USENIX Security) and 3 major software engineering conferences (FSE, ASE, ICSE) containing the keyword "fuzz".

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### **4. Generic Fuzz Testing Model **
The paper formalizes fuzz testing into a generic model (**Algorithm 1**) consisting of two phases:

1. **`PREPROCESS(C)`**: Initial setup before the fuzzing loop (e.g., instrumentation, seed selection, trimming).
2. **Main Iteration Loop**:
   * **`SCHEDULE`**: Selects a configuration for the current iteration based on time budget and history.
   * **`INPUTGEN`**: Generates concrete test cases using seeds, models, or mutation techniques.
   * **`INPUTEVAL`**: Executes PUT on test cases and checks correctness policies via `Bug Oracle`.
   * **`CONFUPDATE`**: Updates the set of fuzz configurations based on execution feedback.
   * **`CONTINUE`**: Evaluates whether to proceed with another iteration.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### **5. Fuzzer Taxonomy **
Fuzzers are categorized by the granularity of internal program semantics observed during execution:
* **Black-box Fuzzer**: Inspects only I/O behaviors without examining internal program structure.
* **White-box Fuzzer**: Analyzes internal code structures and execution states (e.g., Dynamic Symbolic Execution / Concolic Testing, Taint Analysis); achieves high precision but incurs heavy computational overhead.
* **Grey-box Fuzzer**: Collects lightweight execution feedback (e.g., code/branch coverage) without full semantic analysis, balancing execution speed and exploration efficiency.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### **6. Fuzzer Lineage & Preprocessing **
* **Genealogy (Figure 1, p. 5)**: Maps fuzzer development chronologically from Miller et al. (1990) across black/grey/white-box categories and target domains (File, Network, Web, Kernel, Concurrency, UI).
* **Preprocessing Tasks **:
  * **Instrumentation**: Static (compile-time) or Dynamic (runtime) code insertion to collect execution feedback.
  * **In-Memory Fuzzing**: Uses memory snapshots or persistent API calls to bypass expensive process re-initialization.
  * **Seed Selection & Trimming**: Applies minset algorithms to select minimal seed sets that maximize coverage, and trims individual seed file sizes to increase execution throughput.
  * **Driver Preparation**: Prepares wrapper drivers to target isolated components like libraries or kernel APIs.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
