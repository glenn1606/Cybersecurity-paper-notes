# Paper Summary: What You Corrupt Is Not What You Crash (WYCINWYC)

**Paper:** *What You Corrupt Is Not What You Crash: Challenges in Fuzzing Embedded Devices*
**Authors:** Marius Muench, Jan Stijohann, Frank Kargl, Aurélien Francillon, Davide Balzarotti (EURECOM, Siemens, Ulm University)
**Venue:** NDSS 2018
**Code:** https://github.com/avatartwo/ndss18_wycinwyc

> **Context for myself:** this is not a fuzzer or an LLM paper — it's a **measurement study** of why fuzzing embedded devices under-detects bugs, plus a set of runtime heuristics to fix that. I'm reading it as background for my planned research on *LLM-assisted fuzzing (seed/harness generation) of closed-source ECHONET Lite devices*, because whatever seeds or harness I build, the question "did the device actually break, or did it just look fine?" applies directly to a real EL device over the network. This note focuses on lessons, not on re-explaining the paper.

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

Fuzzing assumes that a memory corruption is eventually *visible*: a crash, a segfault, a core dump. That assumption holds on desktop systems because of layers of protection (MMU, heap integrity checks, stack canaries, sanitizers). It does **not** generally hold on embedded devices, which often lack these mechanisms due to cost and resource constraints.

The paper calls this a **silent memory corruption**: the memory is corrupted, but the program keeps running with no visible symptom, until (maybe) much later, if ever. If a fuzzer relies only on "does the device still respond", it will miss most of these.

They identify three general challenges of fuzzing embedded systems:

- **C1 Fault Detection**: no MMU-driven segfaults, no heap canaries, minimal or no I/O to report an error even when one occurs.
- **C2 Performance and Scalability**: can't easily get many parallel copies of a physical device, and resetting state often means a full reboot (can take minutes).
- **C3 Instrumentation**: coverage-guided fuzzing and sanitizers need source or recompilation, which is rarely available for proprietary embedded firmware.

## How does it work? (short version)

**Device classes.** They split embedded systems into three types by OS sophistication, because that's what actually determines what crash-detection machinery exists:

- **Type-I**: general-purpose OS retrofitted for embedded use (embedded Linux + busybox/uClibc). Has an MMU in their test device.
- **Type-II**: embedded OS (uClinux, VxWorks, ZephyrOS) — kernel/application separation exists, but often **no MMU**.
- **Type-III**: no OS abstraction at all, a single monolithic control loop (common in small microcontrollers, e.g. Cortex-M).

**Controlled experiment.** They inject artificial bugs (stack/heap buffer overflow, double free, null pointer dereference, format string) into two real libraries (`expat`, mbed TLS), each with an independent trigger condition (an idea borrowed from LAVA). They run the *same* bugs on a desktop Linux, a Type-I router, a Type-II IP camera, and a Type-III dev board, and record what actually happens using **six outcome categories**: Observable Crash, Reboot, Hang, Late Crash, Malfunctioning, No Effect. Only the first two are fuzzer-friendly; the rest range from "annoying but detectable" to "the worst possible case — totally silent".

**Mitigations.** Six engineering options are laid out with their prerequisites (static instrumentation, binary rewriting, physical re-hosting, full emulation, partial emulation, hardware tracing), each with clear trade-offs. They then build and evaluate **six runtime heuristics** that each mimic a specific desktop protection mechanism in software, implemented as plugins on top of Avatar (orchestrates partial/full emulation) + PANDA (the emulator/analysis engine):

- Segment Tracking (mimics MMU segfaults)
- Format Specifier Tracking (mimics printf format-string hardening)
- Heap Object Tracking (mimics AddressSanitizer for heap)
- Call Stack Tracking (mimics shadow stacks)
- Call Frame Tracking (coarse stack-overflow detection)
- Stack Object Tracking (fine-grained stack AddressSanitizer equivalent)

Each heuristic needs different information (executed instructions, register state, memory accesses, memory map, or debug annotations — Table IV lays this out explicitly) and they're explicitly designed to work both live (in an emulator) and "post-mortem" on a saved execution trace.

## Results

- With **no heuristics**, a plain liveness check (does the device still answer as expected) detected only 2 of 5 injected vulnerability classes (format string, stack overflow) — because those happened to cause a hang. Heap overflow, double free, and null pointer dereference were **completely invisible** to a liveness check across the embedded platforms tested.
- With **all heuristics combined**, 100% of the injected corruptions were detected, across all four fuzzing configurations (native device, partial emulation with memory forwarding, partial emulation with peripheral modeling, full emulation).
- Only one **false positive** was observed across their whole experiment (an over-approximation in stack object tracking on two adjacent variables).
- **Performance cost**: partial emulation with memory forwarding (PE/MF) was **more than an order of magnitude slower** than the native device, because of the I/O round-trip to the real hardware over JTAG/USB. Partial emulation with peripheral modeling (PE/PM) and full emulation (FE) were between 1.5× and 6× slower *with heuristics on*, but — critically — **full emulation with all heuristics enabled still processed more inputs per hour than fuzzing the native, unprotected device**, while detecting every corruption the native device missed.
- The device classification itself predicts detection: desktop Linux crashed every time; the embedded Linux (Type-I) sometimes delayed the crash (e.g. heap errors only surfaced in the `exit()` handler); the no-MMU systems (Type-II, Type-III) very rarely crashed observably at all.

## What's different from previous work?

| Previous work | Limitation | WYCINWYC |
|---|---|---|
| Fuzzing studies on specific domains (automotive CAN, GSM phones, smart cards, PLCs — Table I in the paper) | Each reports fuzzing worked to some extent, but none independently studies *why* crash detection on embedded devices is unreliable | First dedicated, cross-platform study of silent corruption as a general phenomenon |
| PROSPECT (partial emulation + I/O forwarding for Type-I) | Improves testability but doesn't address fault *detection* once the firmware is running | Adds detection heuristics on top of the same partial-emulation idea |
| AddressSanitizer, shadow stacks, CFI (desktop techniques) | Need source + compiler toolchain, essentially never available for closed firmware | Re-implements the *idea* of each technique as an external, non-intrusive runtime monitor, no recompilation needed |
| Avatar / Surrogates (orchestration frameworks) | Solve the "how do I run this firmware at all" problem | Reuses Avatar's orchestration, but adds the missing "how do I know something went wrong" layer |

The core contribution isn't a new fuzzer — it's demonstrating, with a controlled experiment, that an *assumption baked into almost every fuzzing tool* (visible crash = bug found) is frequently false on embedded targets, and then showing a concrete, measured way to patch that assumption using emulation-based monitoring.

## Can it be improved? (my thoughts)

- **Artificial bugs only, by design.** The authors are upfront about this: they inserted bugs with independent trigger conditions (LAVA-style) rather than studying real, naturally-occurring vulnerabilities. This is reasonable for isolating the "does the platform make this visible" variable, but it means the false-positive/false-negative rates they report for the heuristics are explicitly **not** meant to generalize — they say so themselves. A reader citing "100% detection" should be careful to note it's 100% of *their inserted* bugs in *this* setup.
- **One device per class.** Conclusions about Type-I/II/III rest on a single representative device each (one router, one IP camera, one Nucleo board). Real device diversity within a class (different MMUs, different libc hardening, different watchdog behavior) could change the numbers meaningfully.
- **Heuristics need a working (partial or full) emulator.** The paper itself names full emulation as "still very difficult to achieve" and partial emulation as paired with a physical device requirement and a serious throughput hit. For a device with no public emulator and no datasheet-level documentation, none of their six heuristics can actually be deployed as described.
- **Heuristics assume access to the firmware's binary semantics.** Several heuristics need allocator function locations, format-string function locations, or debug symbols for stack layout. For a genuinely black-box, closed-source target with firmware you can't disassemble easily, recovering this information is itself a hard reverse-engineering problem, not something free.
- **No exploration of application-level "malfunctioning" (R5) detection.** The paper explicitly names this as the hardest case (wrong output with no crash) and says a fuzzer would need to know the correct output for each input — then essentially leaves it unsolved, suggesting only "insert known-good functional test requests between fuzz inputs" as a costly workaround. This is exactly the gap that a spec-driven oracle (like the ones in mGPTFuzz or ChatAFL) is meant to fill, but WYCINWYC doesn't attempt it.
- **Single serial-protocol target for the quantitative experiments.** All throughput and detection numbers come from one `expat`-based Type-III target over a simple textual serial protocol. It's a reasonable worst case (as they argue), but the quantitative numbers (1.5–6× slowdown, >10× for PE/MF) are specific to this setup and may not transfer directly to a networked protocol like ECHONET Lite over UDP.

## Lessons and experience I take from this project

These are the points I want to remember for my own project, not just for this paper.

1. **"No crash" is not evidence of "no bug" — ever, but especially on embedded targets.**
   This is the single biggest takeaway. If my ECHONET Lite fuzzer only watches for the device going offline or timing out, I will likely miss most memory-corruption-class bugs, because EL devices are exactly the kind of resource-constrained, often-no-MMU targets this paper studies. I need an explicit plan for detecting **R5-style malfunctioning** (device responds, but wrongly) and not just R1/R2 (crash/reboot).

2. **A device's "class" (Type-I/II/III) is a cheap, useful predictor of how hard fault detection will be, and I should figure out the device's class before I even think about seed generation.**
   Before building seeds and harness for a given closed-source EL device, I should try to determine what's running underneath (full Linux, a lightweight RTOS, or a bare-metal loop), since that tells me up front whether I can expect any crash visibility at all, or whether I need to design in extra liveness/correctness probing from day one.

3. **A spec-aware oracle is my best lever against the "malfunctioning" problem, and this paper validates why it matters.**
   WYCINWYC names the malfunctioning case as the hardest to detect and basically punts on it. My planned approach — using the EDO spec's declared property types, value ranges, and access rules (Get/Set/Observable) as ground truth for what a *correct* response should look like — is directly aimed at this exact gap. I should treat "detecting wrong-but-not-crashed responses" as a first-class design goal, not an afterthought, because off-the-shelf fuzzing assumptions won't catch it for me.

4. **Active probing (known-good requests interleaved with fuzz input) is a legitimate, if costly, technique — and I can budget for it deliberately.**
   The paper's workaround for malfunctioning-detection is to periodically send requests with a known-good expected answer and compare. For EL, this maps cleanly: after a batch of fuzzed Set/Get requests to a property, send a plain Get on a stable property (or the same one) and check the response against the spec. I should design this as an explicit, tunable part of my fuzzing loop rather than discovering I need it after the fact.

5. **Separate "is the target even alive" from "is the correctness heuristic catching things" — track both, and know their detection overlap.**
   Table V and Figure 1 in the paper explicitly break down which heuristic caught which bug class, and show the liveness check's detections are a *subset* of what the combined heuristics catch (segment tracking alone supersedes the liveness check's hits). For my evaluation, I should similarly report, per bug/anomaly found: was it caught by a simple "device still responds" check, or only by comparing the actual response against the spec-derived expected one? That distinction is the evidence for whether my spec-aware oracle adds real value over a naive approach.

6. **Decide early whether full emulation, partial emulation, or the real device is realistic for the target — and budget time accordingly.**
   The paper is blunt that full emulation is "still very difficult to achieve" even with research effort, and partial emulation costs an order of magnitude in throughput. For closed-source EL devices, I most likely won't have firmware images or datasheet-level hardware docs at all, which probably rules out both emulation paths the way this paper implements them. That means I should plan around **testing the real device over the network**, and build my correctness oracle and monitoring around what I can observe externally (EL responses, timing, side channels like UPnP/mDNS liveness if available) rather than assuming I'll get emulator-level introspection.

7. **A controlled, synthetic experiment is a legitimate and efficient way to validate a detection method before trusting it on real bugs.**
   Rather than hunting for and depending on naturally occurring vulnerabilities to validate their heuristics, the authors inserted bugs with known, independent trigger conditions, which let them cleanly measure detection rates without relying on luck. If I want to validate my own correctness oracle (does it actually notice when an EL device misbehaves?) before pointing it at real devices, I could do something similar: build a small reference EL server/simulator where I deliberately inject a few spec violations (wrong value returned after Set, wrong error code for an invalid property) and check whether my oracle catches them, before trusting it on hardware.

8. **State-reset cost is a real engineering line item, not a footnote.**
   They explicitly measure that physical device reboots can take minutes and that this dominates fuzzing throughput at higher corruption-trigger probabilities (`PC`), motivating their snapshot/restore trick in the emulator. For a physical EL device I can't snapshot, I should plan for and measure how long a device takes to recover after a bad request (does it need a power cycle, or does ECHONET's instance/EPC model let me probe aliveness cheaply?), since this directly determines how many test cases per hour are realistically achievable.

9. **Report false positive/negative behavior honestly and explicitly flag what's implementation-specific versus fundamental.**
   The paper is careful to say their false-positive/negative rates are "highly target- and implementation-dependent" and that a full FP/FN study was out of scope — they only report the one false positive they actually observed. When I report how well my spec-based oracle distinguishes "device behaved correctly" from "device misbehaved", I should be equally careful not to overclaim generality from results on one or two device classes.

10. **A "worst case" test target is a deliberate and useful methodological choice.**
    They picked the Type-III (no OS, no MMU) device for their quantitative experiments specifically because it's the hardest case, reasoning that if detection works there, it's likely to work on the easier Type-I/II cases too. When I pick which ECHONET Lite device classes to prioritize for testing, I should think similarly: is there a "hardest case" device (least capable, most resource constrained) that would make my results more convincing if the approach works on it?

## How this connects to my ECHONET Lite fuzzing idea

- **Fault detection is a separate problem from input generation, and I need both.** My planned work is mostly about *what to send* (LLM-guided seeds/harness from the EDO spec). This paper is a hard reminder that *how do I know the device broke* is an equally important, separate problem that I cannot assume away. A great seed generator pointed at a device with no visible crash behavior still produces zero useful results if I have no way to detect that something went wrong.
- **Spec-as-oracle is my answer to the "malfunctioning" gap this paper leaves open.** Combined with what I read in mGPTFuzz (spec-derived FSMs used as oracle for non-crash bugs) and ChatAFL (structure/state-aware generation), this paper gives me the "why this matters" framing: on resource-constrained embedded targets, crash-based detection alone is known to be unreliable, so a correctness oracle derived from the EDO spec (declared types, value ranges, Get/Set/Observable rules) isn't a nice-to-have, it's close to necessary.
- **No emulation path realistically available for closed-source EL devices.** Unlike FIRM-AFL (which assumes Firmadyne-emulatable Linux firmware) or this paper's own heuristics (which need an emulator or hardware trace port), I should plan my ECHONET Lite project around black-box, over-the-network testing of the real device, with correctness checking done purely from the outside (EL responses, timing, and whatever liveness/side-channel signals the protocol and device expose).
- **Gap this paper leaves open that's relevant to me:** it studies memory-corruption-class bugs almost exclusively (buffer overflow, UAF-adjacent, double free, format string, null deref) — not application-logic bugs. ECHONET Lite devices plausibly have both: low-level memory bugs in their EL stack implementation, and higher-level logic bugs (wrong property semantics, broken access-rule enforcement) that this paper doesn't address at all but that mGPTFuzz-style spec oracles target directly.

## Conclusion / what I learned

WYCINWYC isn't about a clever new fuzzing algorithm — it's a rigorous demonstration that a basic assumption most fuzzing work makes (crashes are visible) breaks down exactly in the domain (embedded/IoT) that increasingly matters most for security. The methodology — classify targets by what detection infrastructure they plausibly have, inject controlled bugs to measure detection rates cleanly, then engineer and measure mitigations with real performance numbers — is a template I want to borrow regardless of whether I ever touch emulation myself.

**For my own fuzzing learning journey:** this paper reframed "did it crash" as just one noisy signal among several possible outcomes (crash, reboot, hang, late crash, malfunction, no effect), and made me realize that for ECHONET Lite — where I almost certainly won't have an emulator or firmware image for a closed-source device — my entire strategy for *detecting* that something went wrong has to be built from what's observable over the network and from the spec's own declared correctness rules, not inherited for free from the platform. That single realization changes how I think about the "oracle" part of my planned research: it's not a supporting detail, it's possibly the hardest and most important part to get right.
