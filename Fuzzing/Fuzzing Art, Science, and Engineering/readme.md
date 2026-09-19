**Paper Title:** The Art, Science, and Engineering of Fuzzing: A Survey

**Authors:** Valentin J.M. Manès, Hyung Seok Han, Choongwoo Han, Sang Kil Cha, Manuel Egele, Edward J. Schwartz, Maverick Woo

**Year:** 2019 (Submitted/Revised 2019, IEEE TSE 2021)

**Publisher / Venue:** IEEE Transactions on Software Engineering (IEEE TSE) / IEEE

---

## 1. The problem the paper tackles

- Fuzzing has exploded (1000+ GitHub repos, many top-venue papers), but the knowledge is **fragmented**.
- Many fuzzers are documented only by source code + man page → design decisions get lost.
- **Terminology is inconsistent.** Example: the same "shrink a crashing input" idea is called
  - *test case minimization* (AFL)
  - *test case reduction* (funfuzz)
  - while BFF's *crash minimization* means something different (minimize bits that differ from the original seed).
- The last big references were 3 trade-books from 2007–2008, which predate AFL-era progress.

**Goal:** give the field a *common vocabulary*, *one general model* that fits every kind of fuzzer, and a *taxonomy* of the literature.

---

## 2. Core definitions (worth memorizing)

| Term | Meaning |
|---|---|
| **PUT** | Program Under Test |
| **Fuzzing** | Running the PUT with inputs sampled from a *fuzz input space* that **protrudes** the expected input space (contains at least one unexpected input) |
| **Fuzz testing** | Using fuzzing to check if the PUT violates a *correctness policy* |
| **Fuzzer** | Program that does fuzz testing |
| **Fuzz campaign** | One run of a fuzzer on a PUT with a specific policy |
| **Bug oracle** | Decides whether an execution violates the policy |
| **Fuzz configuration** | Parameters controlling the fuzz algorithm (e.g. `(PUT, seed, mutation ratio)` in BFF) |
| **Seed / seed pool** | Well-formed inputs that get mutated; the pool may evolve |

Small but important remarks:
- Sampling is **not necessarily random** (e.g. white-box fuzzers use solvers).
- Policies can be anything **EM-enforceable** (observable in one execution), not just "crash".
- Fuzz-inspired ideas are used beyond security, e.g. PerfFuzz (performance bugs), DeepXplore (neural nets).

---

## 3. The model fuzzer (Algorithm 1), the main contribution

```
B ← ∅
C ← PREPROCESS(C)
while t_elapsed < t_limit ∧ CONTINUE(C):
    conf            ← SCHEDULE(C, t_elapsed, t_limit)
    tcs             ← INPUTGEN(conf)
    B', execinfos   ← INPUTEVAL(conf, tcs, O_bug)
    C               ← CONFUPDATE(C, conf, execinfos)
    B               ← B ∪ B'
return B
```

| Function | What it does | Typical examples |
|---|---|---|
| **PREPROCESS** | Set up before the loop | Instrumentation, seed selection (minset), seed trimming, writing a driver |
| **SCHEDULE** | Pick which configuration to fuzz next | AFL queue, AFLFast power schedule, MAB in Woo et al. |
| **INPUTGEN** | Turn a config into concrete test cases | Bit-flip, grammar-based, symbolic execution |
| **INPUTEVAL** | Run PUT + check bug oracle | Sanitizers, fork server, triage |
| **CONFUPDATE** | Update the config set using run feedback | Add seeds with new coverage, cull seeds, update model |
| **CONTINUE** | Decide whether to keep going | Mostly matters for white-box (stops when no paths left) |

Why it's useful: any fuzzer can be described by *which of these five functions it implements and how*. Radamsa, for instance, has `CONFUPDATE` return C unchanged. Table 1 in the paper classifies ~60 fuzzers this way.

---

## 4. Taxonomy: black / grey / white box

Classified by **how much semantic information the fuzzer observes per run** (not the same as traditional testing, which only has black/white).

- **Black-box**: only sees input/output. Fast, simple (zzuf, Radamsa, BFF, Peach, Miller's original fuzz). Some use input structure (Peach, funfuzz) but still don't look inside the PUT.
- **White-box**: analyses PUT internals → *dynamic symbolic execution* (DSE / concolic) or taint analysis. Systematic but slow (SAGE, KLEE, Mayhem-style).
- **Grey-box**: lightweight/approximate info, usually **code coverage** (AFL, LibFuzzer, honggfuzz, VUzzer). Trades precision for speed → more executions.
- **Hybrid**: alternate grey- and white-box (Driller, QSYM, DigFuzz, Cyberdyne).
- Boundaries are fuzzy; authors admit they used their own judgment in Table 1.

---

## 5. Stage-by-stage notes

### 5.1 PREPROCESS (§3)
- **Instrumentation**
  - *Static* (compile time, low overhead, libs must be recompiled) vs *dynamic* (Pin, DynamoRIO, Valgrind, QEMU; easier for shared libs, higher overhead).
  - AFL supports both (compiler pass or QEMU mode).
  - Coverage info: AFL stores edge coverage in a compact bit vector → **path collisions** cause inaccuracy. CollAFL fixed this with a new hash.
  - Instrumentation can also control **thread scheduling** to trigger race conditions.
- **In-memory fuzzing**: snapshot after slow init (e.g. GUI startup), restore + inject each test case. *In-memory API fuzzing* (AFL persistent mode, LibFuzzer) loops on a function without resetting state → **unsound** (crashes may not reproduce: invalid calling context, hidden side effects).
- **Seed selection**: the *minset* problem. Find the smallest set of seeds that keeps the same coverage. Example from the paper: `s3 → {10,20,30}` beats `s1 → {10,20}` + `s2 → {20,30}`. Backed by Miller: +1% coverage ⇒ +0.92% bugs found.
- **Seed trimming**: shrink seeds while keeping coverage (AFL) → less memory, faster runs. Rebert et al.: "smallest-first" seed selection actually found *fewer* unique bugs than random.
- **Driver preparation**: needed when the PUT can't be fuzzed directly (libraries, kernels, IoT). Largely manual.

### 5.2 SCHEDULE (§4)
- Core issue = the **Fuzz Configuration Scheduling (FCS) problem**: *exploration vs. exploitation*. Spend time learning about configs, or fuzz the ones that look best?
- **Black-box FCS**
  - Householder & Foote: treat fuzzing as Bernoulli trials; favor configs with higher (unique crashes / runs) → more crashes in BFF.
  - Woo et al.: model as *Weighted Coupon Collector's Problem with Unknown Weights*, use **multi-armed bandits**, normalize by time spent (favor fast configs), and make iterations time-based rather than run-count-based.
- **Grey-box FCS** (AFL = evolutionary algorithm)
  - AFL "favorites" = smallest + fastest input per edge; circular queue; gives more runs to fast, high-coverage seeds.
  - **AFLFast**: prioritize rare/new paths; **power schedule** (energy starts small, grows exponentially, normalized by how often that path is exercised) → less time on frequently hit paths.
  - Follow-ups: **AFLGo** (directed to target locations), **Hawkeye** (static analysis for directed fuzzing), **FairFuzz** (target rare branches via mutation masks), **QTEP** (prioritize "faulty" code).

### 5.3 INPUTGEN (§5)
**Model-based (generation)**
- *Predefined model*: user gives a spec/grammar (Peach, Dharma, Tavor with EBNF, Sulley/SPIKE APIs). Kernel fuzzers use syscall templates (Trinity, syzkaller). Language-specific ones: jsfunfuzz, LangFuzz, cross_fuzz, DOMfuzz.
- *Inferred model*: learn the format/protocol
  - In PREPROCESS: Skyfire (probabilistic grammar), IMF (kernel API model from logs), CodeAlchemist ("code bricks" + assembly constraints), Neural fuzzer, Learn&Fuzz.
  - In CONFUPDATE (online): PULSAR (protocol state machine), GLADE (context-free grammar), go-fuzz.
- *Encoder model*: MutaGen mutates the **encoder program** (via program slices) to produce slightly malformed outputs.

**Model-less (mutation)**
- Pure random testing is hopeless: `if (input == 42)` → 1/2³² chance; random bytes won't form a valid MP3. So mutate valid seeds.
- Mutation techniques:
  - **Bit-flipping** with a *mutation ratio* K/N. SymFuzz showed the best ratio is PUT-dependent; BFF/FOE try an exponentially-scaled set of ratios.
  - **Arithmetic**: treat bytes as an integer, add/subtract a small r (AFL default 0 ≤ r < 35).
  - **Block-based**: insert / delete / replace / permute / append / splice between seeds.
  - **Dictionary-based**: "interesting" values (0, -1, 1, `%s`, `%x`, Unicode).

**White-box**
- **DSE / concolic**: build a path formula per path, solve with SMT to get concrete inputs. Slow, so limit its use (Chopper) or combine with grey-box (Driller, QSYM, DigFuzz, which uses grey-box to estimate which paths are hard).
- **Guided fuzzing** (costly analysis first, then cheaper fuzzing): TaintScope ("hot bytes"), Dowser (loops with pointer derefs), VUzzer, GRT, **Angora** (taint + gradient-descent-like search), **RedQueen** (input-to-state correspondence: match comparison operands to input bytes).
- **PUT mutation**: bypass checksums by *patching the program*. TaintScope; **T-Fuzz** removes "Non-Critical Checks" and later reconstructs the crash on the original PUT with symbolic execution.

### 5.4 INPUTEVAL (§6)
- **Bug oracles.** The default "did it crash (fatal signal)?" misses silent memory corruption, so use **sanitizers**:
  - **ASan** (spatial + temporal errors, shadow memory, ~73% slowdown); MEDS (big red-zones on 64-bit).
  - **SoftBound/CETS**: complete in theory, ~116% overhead.
  - **CaVer / TypeSan / HexType**: bad C++ casts.
  - **CFI**: illegal control-flow transitions.
  - **MSan** (uninitialized memory, ~150%), **UBSan** (undefined behavior), **TSan** (data races).
  - Higher-level oracles: KameleonFuzz (XSS via real browser DOM), µ4SQLi (SQLi via DB proxy), **differential testing** (NEZHA, classfuzz, Frankencerts), Jung et al. (info leaks).
- **Execution optimizations**: AFL **fork server**; in-memory fuzzing; Xu et al.'s new syscall to replace `fork()`.
- **Triage** = deduplication → prioritization → minimization.
  - *Deduplication*
    - **Stack backtrace hashing** (top n frames; n varies 1, 3, 5, unlimited; some tools use major + minor hash). The assumption "similar stack = same bug" was **never directly tested**, and heap corruption often crashes far from the real bug.
    - **Coverage-based** (AFL: crash is unique if it hits a new edge, or misses an edge present in all earlier crashes).
    - **Semantics-aware**: RETracer (reverse data-flow from crash dump to "blame" a function), Van Tonder et al. (program repair to map crashes to bugs).
  - *Prioritization / exploitability*: `!exploitable` (EXPLOITABLE > PROBABLY_EXPLOITABLE > UNKNOWN > NOT_LIKELY_EXPLOITABLE), GDB `exploitable`, CrashWrangler. Rule-based heuristics, **not systematically validated**.
  - *Minimization*: delta debugging, Lithium, CReduce (C-specific, faster because it knows the grammar), BFF (min. bit differences), AFL (zero out bytes + shorten). Differs from seed trimming because it can use the **bug oracle**.

### 5.5 CONFUPDATE (§7)
- Black-box: usually returns C unchanged (BFF's "crash recycling" is an exception).
- **Evolutionary seed pool update**: add a test case to the pool if it finds new coverage (fitness function). Refinements:
  - AFL: hit-count buckets.
  - **LAF-INTEL**: split multi-byte comparisons so progress on partial matches counts (also LibFuzzer, honggfuzz, go-fuzz, Steelix).
  - Angora: include calling context.
  - VUzzer: weight basic blocks (rare normal blocks good; error-handling blocks negative).
  - STADS: ecology-inspired estimate of how much is still undiscovered.
  - DeepXplore: "neuron coverage".
- **Maintaining a minset**: cull to avoid pool explosion. Cyberdyne removes non-minset seeds; **AFL only marks them "favorable"** for a higher pick chance ("balance between queue cycling speed and diversity").

---

## 6. What I learned (takeaways)

1. **Fuzzing = one loop, five knobs.** Every fuzzer is just a choice of preprocess / schedule / generate / evaluate / update. Comparing tools becomes much easier with this lens.
2. **"Color" of a box is about feedback granularity**, not about whether you have source. Grey-box wins in practice because speed of *many cheap runs* beats *few smart runs*, and hybrids try to get both.
3. **Scheduling is a bandit problem in disguise.** AFLFast's power schedule and Woo et al.'s MAB approach are the same explore/exploit trade-off applied to seeds.
4. **Random mutation can't get past "magic" checks** (`== 42`, checksums, structured formats). Solutions form a ladder:
   - split comparisons (LAF-INTEL) →
   - match operands to input (RedQueen) →
   - taint + gradient search (Angora) →
   - symbolic execution (Driller/QSYM) →
   - patch the checks out (T-Fuzz/TaintScope).
5. **Finding a crash isn't enough.** A fuzzer needs a good *oracle* (sanitizers catch bugs that don't crash) and good *triage* (dedup, exploitability, minimization). A fuzzer can report thousands of crashes that map to a handful of bugs.
6. **Seed quality matters as much as the algorithm**: minset, trimming, culling, and preprocessing all shape the campaign.
7. **Practical engineering matters**: fork server, persistent mode, snapshotting, and compact coverage maps aren't glamorous but drive throughput.
8. **Trade-offs to remember**
   - In-memory API fuzzing: fast but unsound.
   - ASan: cheap; SoftBound/CETS: complete but slower.
   - Static instrumentation: fast; dynamic: handles libs and binaries.
   - Model-based: deeper inputs but needs a model; mutation-based: no model but shallow.

---

## 7. Limitations / open issues (from the paper + my reading)

- Pure **survey**, no experiments, so no ranking of which technique works best. (Klees et al. "Evaluating Fuzz Testing" covers evaluation; the paper calls it orthogonal.)
- Deliberately **excludes most DSE work** unless the authors call it fuzzing → scope depends on the word "fuzz".
- Grey/white/black classification is partly **subjective** (admitted).
- Unvalidated assumptions the paper points out:
  - stack-hash dedup hypothesis,
  - exploitability heuristics (`!exploitable` etc.).
- Written in 2019: newer work (ML-guided fuzzing, more hybrid/directed fuzzing, etc.) isn't covered.

---

## 8. Quick glossary

- **Concolic testing**: concrete + symbolic execution together.
- **Minset**: minimal seed set that keeps the same coverage.
- **Power schedule**: how many mutations a seed gets per selection.
- **Persistent mode**: AFL's loop that reuses one process for many inputs.
- **Fork server**: pre-initialized process that forks per test case.
- **Sanitizer**: compile-time instrumentation that turns silent bugs into detectable aborts.
- **NCC**: Non-Critical Check (T-Fuzz).
- **Shadow memory**: metadata map used by ASan/MSan to track validity/initialization.