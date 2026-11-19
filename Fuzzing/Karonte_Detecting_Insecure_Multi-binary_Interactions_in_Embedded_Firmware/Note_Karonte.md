# Paper Summary: KARONTE

**Paper:** *KARONTE: Detecting Insecure Multi-binary Interactions in Embedded Firmware*
**Authors:** Nilo Redini, Aravind Machiry, Ruoyu Wang, Chad Spensky, Andrea Continella, Yan Shoshitaishvili, Christopher Kruegel, Giovanni Vigna (UC Santa Barbara, Arizona State)
**Venue:** IEEE S&P 2020
**Code:** https://github.com/ucsb-seclab/karonte

> **Context for myself:** this is a **static analysis** paper, not fuzzing. I'm reading it as background for my planned research on *LLM-assisted fuzzing (seed/harness generation) of closed-source ECHONET Lite devices*, because it deals with the same underlying problem I'll face: a device's functionality is split across multiple programs, and a single component looked at alone gives a misleading picture of what an attacker can actually reach. This note is shorter on "what the paper does" and longer on "what I can learn from it".

## Table of Contents

- [What problem does it solve?](#what-problem-does-it-solve)
- [How does it work? (short version)](#how-does-it-work-short-version)
- [Results](#results)
- [What's different from previous work?](#whats-different-from-previous-work)
- [Can it be improved?](#can-it-be-improved-my-thoughts)
- [Lessons and experience I take from this project](#lessons-and-experience-i-take-from-this-project)
- [How this connects to my ECHONET Lite fuzzing idea](#how-this-connects-to-my-echonet-lite-fuzzing-idea)
- [Conclusion / what I learned](#conclusion--what-i-learned)

---

## What problem does it solve?

Embedded firmware (routers, IoT devices) is rarely one program. A typical setup: a web server accepts an HTTP request, hands data to a CGI handler binary through environment variables or sockets, and that handler may call yet another program. Each binary makes its own assumptions about the data it receives.

This creates a dilemma for static analysis:

- **Analyze each binary alone** → the analysis has no idea which inputs are actually attacker-controlled, so it must assume every input source is hostile. In the paper's own numbers, single-binary analysis on NETGEAR firmware produced an average of **729 alerts** per sample, most of them false positives, and verifying all of them would take an analyst about **138 hours** per firmware image (or about 4 months across their whole dataset).
- **Analyze only the network-facing binary** → fast and low on false positives, but misses real bugs that live deeper in the call chain, in binaries that never touch the network directly.

So the goal: track data **across binary boundaries** to know which inputs are really attacker-controlled by the time they reach a dangerous sink, without emulating the whole firmware (which is slow and often fails, as FIRM-AFL and Firmadyne already show).

## How does it work? (short version)

KARONTE has five stages:

1. **Unpack** the firmware with `binwalk`.
2. **Border Binaries Discovery**: find which binaries parse network input. It scores functions using features borrowed from prior parser-detection work (number of basic blocks, branches, comparisons) plus two new ones: a *network mark* (comparisons against known protocol strings like "soap" or "HTTP") and a *connection mark* (does data from a socket reach a comparison at all). Binaries are clustered (DBSCAN) and the cluster containing the highest-scoring binary becomes the initial "border binary" set.
3. **Binary Dependency Graph (BDG)**: starting from where attacker data enters a border binary, a taint engine explores paths and calls a set of **Communication Paradigm Finders (CPFs)** — one per IPC mechanism (files, shared memory, environment variables, sockets, command-line args) plus a generic OS-independent "Semantic CPF" that just looks for hard-coded values used as data keys. Each CPF that matches tells KARONTE: here is a data key, here is who sets it, here is who might get it, and here are other binaries using the same key (which get added to the analysis). This builds a graph of which binaries talk to which, through what channel.
4. **Multi-binary Data-flow Analysis**: walk the BDG and, instead of enumerating every possible path (path explosion), keep only the **least restrictive set of constraints** seen along the way for a given piece of data. This produces the **Binary Flow Graph (BFG)**, basically the BDG annotated with "how much can this value actually be, once every binary along the chain has applied its own checks".
5. **Insecure Interactions Detection**: walk the BFG and flag places where attacker-controlled data (per the BFG's constraints) reaches a `memcpy`-like sink with a buffer too small, or controls a loop condition entirely (DoS).

Two taint-engine refinements matter:

- **Path prioritization**: when exploring a function symbolically, favour paths likely to propagate taint *out* of the function (e.g. through a `return`), because naive exploration can get lost in paths that never carry the tainted value forward (the "undertaint" problem).
- **Taint tag dependencies**: if a sanitizer function isn't analyzed in full (e.g. `strlen` called but not followed), a derived variable can get a different taint tag than the original, and an untaint policy might accidentally stop tracking the original value while only clearing the derived one. KARONTE links tags so that untainting one also untaints what depended on it.

## Results

- 53 recent firmware images (NETGEAR, D-Link, TP-Link, Tenda, Huawei, Nvidia, Qualcomm) plus a 899-sample dataset from Firmadyne's collection for scalability testing
- **87 alerts total**, **51 true positives** (34 of them multi-binary bugs), **46 zero-days**, plus 5 known bugs re-discovered; 1 CVE and 1 PSV obtained, others confirmed via vendor collaboration or shared-codebase reasoning
- **Alert reduction: 20,931 → 74** when comparing plain single-binary analysis (`ALL`) against the full pipeline (`KARONTE`) on the same 29-sample subset — about **two orders of magnitude**
- Estimated analyst time: from about 138 hours (NETGEAR, single-binary) down to about **14 minutes**
- On the large 899-sample dataset: 38.7% of firmware implemented network functionality through multiple binaries; average BDG has 5 binaries, 3 of them border binaries; 80% of samples fully analyzed within a day
- Path prioritization raised the fraction of fully-explored taint analyses from 75% to 84% (fewer timeouts)
- An independent researcher at another university reproduced the main table from a Docker image and the released source

## What's different from previous work?

| Previous work | Limitation | KARONTE |
|---|---|---|
| Dynamic taint tracking (various) | Too slow for resource-constrained embedded devices | Static, no runtime overhead on the device |
| Firmadyne / Avatar / SURROGATES (emulation) | Strict assumptions on firmware, or needs real hardware / debug ports | Pure static analysis, no emulation needed |
| IoTFuzzer, RPFuzzer, FIRM-AFL, FirmFuzz (fuzzing) | Need a running/emulated target, input generation is the bottleneck, may miss deep non-network-facing bugs | Doesn't need to run the firmware at all |
| Single-binary static taint analysis (BootStomp and similar) | Each binary treated as isolated → explosion of false positives when input provenance is unknown | Explicitly models and propagates data **across** binaries |
| Costin et al. (static + emulation mix for web interfaces) | Not generic, relies on heuristics, no new memory-corruption bugs | General detection modules, found real zero-days |

The core novelty is really the **BDG/BFG abstraction**: instead of guessing where tainted data might be, KARONTE discovers the actual communication channels between binaries (via CPFs) and only propagates taint along edges it has evidence for, carrying forward the least restrictive constraint it has seen. That's what cuts the false-positive rate without just being less thorough.

## Can it be improved? (my thoughts)

- **Linux/POSIX multi-binary world only.** The attacker model is "network data enters through a border binary", which fits routers and similar Linux-based devices well, but many IoT devices are a single monolithic binary on an RTOS, or communicate through proprietary binary protocols rather than IPC the CPFs recognize.
- **MIPS support is weak** because it rides on `angr`. 14 TP-Link samples produced empty BDGs purely because `angr` couldn't resolve data on MIPS, which is a tooling limitation disguised as a result.
- **Border binary discovery can silently fail.** If a binary's strings are computed at runtime (GOT-relative addressing) rather than stored as literals, the scoring features never fire, and the binary is missed entirely — a false negative with no warning.
- **Alert classification is manual and partly judgment-based.** Classifying 87 alerts into true/false positives, and then 51 bugs into "vulnerability" or "just a bug", took a human analyst and, in many cases, vendor cooperation or device purchases. The "46 zero-days" number depends a lot on this manual process.
- **The least-restrictive-constraint heuristic is a double-edged sword.** It is explicitly designed to be unsound in the direction of fewer false negatives from over-constraining, but it could also propagate a constraint that is *too permissive* if one path through a binary is analyzed incompletely (timeouts limit exploration), silently widening what counts as "attacker-controlled" without flagging that.
- **Firmadyne found zero bugs on the same large dataset**, which the authors use as a point in favour of static analysis, but it's also a reminder that comparing a static tool against a dynamic tool that didn't even run correctly isn't the strongest possible comparison.
- **No fuzzing combination explored.** The paper explicitly lists fuzzing-based tools as related/complementary work, but doesn't try combining the two (e.g., using the BDG/BFG to seed or target a fuzzer), even though that seems like a natural next step — and is exactly the kind of thing ChatAFL/mGPTFuzz-style "spec-guided generation" could plug into.

## Lessons and experience I take from this project

These are the points I want to remember for my own project, not just for this paper.

1. **Don't analyze a component in isolation if it isn't reachable in isolation.**
   The paper's central argument is that single-binary analysis on real firmware produces an unworkable number of alerts, because it can't tell attacker-controlled data from everything else. The same risk applies to protocol fuzzing of one ECHONET Lite service in isolation: if a device splits functionality across multiple local processes (a network daemon plus internal handlers), fuzzing or reasoning about just the front-facing one may both miss bugs deeper in the chain and (if I ever add static checks) produce noise if I don't track what's actually attacker-reachable.

2. **Model the actual communication channels, don't assume a single interface.**
   KARONTE's CPFs are protocol-specific recognizers (files, env vars, sockets, command-line args) plus one generic fallback. For an ECHONET Lite device, the "interface" an LLM-guided fuzzer sees (UDP port 3610, ESV services) might be just the front door; if the vendor's device also talks to a cloud backend, a companion app, or a local IPC mechanism, these are other "data keys" worth knowing about, especially for a closed-source device where I can't just read the source to see how components connect.

3. **Reduce false positives by being evidence-based, not just permissive.**
   The BDG's edges are only created for communication the CPFs found actual evidence of — not assumed at every possible boundary. For seed/harness generation with an LLM, the analogous discipline is: don't let the LLM invent fields, EOJs, or property semantics it has no evidence for in the spec; ground generation in what the machine-readable EDO data actually says (as in the eModelGen paper I read before this one), and treat anything else as a hypothesis to verify, not a fact to act on.

4. **A good "least-restrictive" default is useful when exact information isn't available.**
   KARONTE's constraint propagation always keeps the *least* restrictive set of constraints seen, because the goal is to flag what *could* be exploitable, not to find the exact worst case. This reminds me that for fuzzing, when the spec doesn't fully constrain a value (e.g. a property type note says "manufacturer-defined"), the safer default for seed generation is to try the widest sensible range rather than only the most constrained-looking example value — otherwise real bugs at the edges never get tried.

5. **Separate discovery, propagation and detection into stages, and make each one independently checkable.**
   Border Binaries Discovery, BDG Recovery, BFG construction, and Insecure Interactions Detection are four clearly separated stages, each with its own inputs/outputs and its own evaluation numbers (Table II breaks down alerts and time per stage). This is the same decoupling lesson from eModelGen (Semantic Analyzer vs Generators), just applied to a bigger pipeline. For my fuzzer I should keep spec parsing, seed generation, request sending, and oracle checking as separable stages I can test and report on individually — not one monolithic script.

6. **Quantify the human cost you are saving, with a believable method.**
   "7 minutes per alert, 138 hours for one firmware sample" is a concrete, defensible number because they actually timed their analyst. If I claim an LLM-assisted approach saves engineer time versus manually writing ECHONET Lite seeds/harnesses per device class, I should measure this the same way: time a person (maybe myself) doing it by hand for a few classes, not just assert that automation "saves effort".

7. **Verifying ground truth is expensive and uneven, plan for it.**
   Only 2 of their vulnerable devices were physically available to make a PoC; many "confirmed" bugs relied on vendor cooperation, which some vendors refused, or on "this bug also exists in a firmware sharing the same codebase" reasoning. For closed-source ECHONET Lite devices, I should expect the same problem: I may only have 1–2 physical devices, some vendors won't engage, and I'll need fallback verification strategies (code-base reuse across product lines, cross-checking with a second device if I can get one, being explicit in the writeup about which bugs are PoC-confirmed vs reasoned about).

8. **Static tool limits are often really dependency-tool limits.**
   Several of KARONTE's own gaps (MIPS binaries, GOT-relative strings) are actually `angr` limitations, and the paper is upfront about this rather than hiding it. When I pick libraries for an ECHONET Lite harness (an open-source EL stack, an LLM API, maybe a disassembler if I ever touch firmware binaries), I should document which failures are *my* approach's fault versus a dependency's known limitation, the way this paper does.

9. **Independent reproduction is worth designing for from the start.**
   They shipped a Docker image, the source, the exact dataset, and documentation, and had an outside researcher reproduce their main table. If my eventual tool/paper is going to claim bug counts or coverage improvements, I should build it so someone else (or myself, six months later) could rerun it from a container and get the same seeds, same requests, same alerts.

10. **"Scales well" needs its own experiment, separate from "finds bugs".**
    They ran a dedicated 899-sample scalability study distinct from the 53-sample bug-finding study, with its own complexity metrics (binaries, basic blocks, estimated path counts) and its own performance breakdown per stage. If my fuzzer works great on 2–3 ECHONET Lite device classes, that says nothing about whether it would still work reasonably on 50 classes; I should plan a separate, larger-scale run (even just against many simulated device profiles) to check this before claiming generality.

## How this connects to my ECHONET Lite fuzzing idea

- **Attacker model parallel.** KARONTE assumes attacker data enters through a network-facing binary and may affect other binaries indirectly. For ECHONET Lite, a device's EL responder is the "border binary" equivalent; if the device internally hands EL-derived data to other local processes (e.g. a cloud sync daemon, a local web UI), those are exactly the kind of multi-component interactions this paper cares about, and my fuzzing should keep in mind that a bug may not surface as a visible EL response at all.
- **Taint/constraint propagation as a mental model for oracle design.** Even without doing static binary analysis myself, the idea of "track the least restrictive constraint on a value as it flows through the system" is a useful way to think about what values an ECHONET Lite fuzzer should try: a property's declared data type and range (from the EDO spec) is the "constraint" a well-behaved device should enforce, and a good oracle checks whether the device actually enforces it (similar to mGPTFuzz's spec-as-oracle idea, but framed here as constraint checking).
- **Complementary, not competing, with fuzzing.** KARONTE's own related-work section lists fuzzing as a separate, complementary technique with its own limitations (input generation, needing a running target). My project sits squarely in that complementary space: blackbox, over-the-network fuzzing of a real closed-source device, which is exactly what static multi-binary analysis *can't* do without the binaries or a working emulator.
- **Gap this paper leaves open:** nothing here is EL- or even IoT-protocol-specific; it's a generic firmware technique evaluated on routers/cameras. It doesn't touch application-layer protocol semantics or seed generation at all, so it doesn't reduce the novelty of an LLM-guided ECHONET Lite fuzzer, but it's a reminder that bugs can live in places a protocol-level fuzzer alone would never reach (the actual D-Link example bug in Section X-G is a buffer overflow in a *second* binary reached only after the first binary's handling of attacker data).

## Conclusion / what I learned

KARONTE is a good example of a paper whose real contribution is a **disciplined abstraction** (the BDG/BFG) that turns an intractable problem (every binary might be attacker-reachable) into a tractable one (only these specific, evidenced data flows are), and a rigorous **evaluation methodology** (separate scalability study, independent reproduction, analyst-time accounting, responsible disclosure). The static-analysis techniques themselves (CPFs, taint tag dependencies, path prioritization) are specific to binary analysis and not something I'll reuse directly for protocol fuzzing, but the *process discipline* around them is very transferable.

**For my own fuzzing learning journey:** this paper sharpened my sense that "where is the attacker-controlled data, and what does it have to pass through before it matters" is a question worth asking explicitly, not just assuming it stops at the protocol responder. It also gave me a second strong example (after eModelGen) of staged pipeline design with per-stage evaluation, and a very concrete way to think about and report the human-time cost an automated approach saves — something I should build into my own evaluation plan from the start rather than add at the end.

**Next steps:**

- [x] Read KARONTE (IEEE S&P 2020) for multi-binary firmware interaction background
- [ ] Read BootStomp (USENIX Security 2017), since KARONTE's taint engine is built on top of it
- [ ] Read *What You Corrupt Is Not What You Crash* (Muench et al., NDSS 2018), cited here and already on my earlier list, to understand firmware fuzzing limitations from another angle
- [ ] Check whether any open-source ECHONET Lite device stacks (uEcho, echonet-lite npm package) expose or hint at multi-process designs worth knowing about before building a harness
- [ ] Think about whether my fuzzing oracle should check not just the EL response but any side effects (logs, secondary services) that a more KARONTE-like "what else consumes this data" mindset would flag
