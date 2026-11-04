# Paper Summary: FIRM-AFL

**Paper:** *FIRM-AFL: High-Throughput Greybox Fuzzing of IoT Firmware via Augmented Process Emulation*
**Authors:** Yaowen Zheng, Ali Davanian, Heng Yin, Chengyu Song, Hongsong Zhu, Limin Sun (IIE-CAS, UC Riverside)
**Venue:** USENIX Security 2019
**Code:** https://github.com/zyw-200/FirmAFL

## Table of Contents

- [What problem does it solve?](#what-problem-does-it-solve)
- [How does it work?](#how-does-it-work)
- [Results](#results)
- [What's different from previous work?](#whats-different-from-previous-work)
- [Can it be improved?](#can-it-be-improved-my-thoughts)
- [Comparison with mGPTFuzz and ChatAFL](#comparison-with-mgptfuzz-and-chatafl)
- [Conclusion / what I learned](#conclusion--what-i-learned)

---

## What problem does it solve?

IoT firmware is a big attack surface (Mirai-style botnets exploit firmware bugs). AFL is the most successful greybox fuzzer, but it can't be applied directly to IoT programs. There are two problems:

1. **Compatibility.** IoT programs depend on special hardware and kernel behaviour (NVRAM, network interfaces, custom drivers). If you extract a program from a firmware image and run it with AFL's user-mode QEMU, it usually fails. In the paper, AFL with user-mode QEMU failed on all their test programs.
2. **Throughput.** Full-system emulation (e.g. Firmadyne) is compatible, but it is about **10× slower** than user-mode emulation. Greybox fuzzing lives or dies by speed (more executions per second means more coverage), so this hurts a lot.

So the goal: get **the compatibility of full-system emulation and the speed of user-mode emulation**, with no modification to the target program.

Where the full-system overhead comes from (the authors profiled `basename` and `uptime`):

| Bottleneck | Cause |
|---|---|
| **B1. Memory address translation** | Software MMU translates every memory access (guest virtual to guest physical to host virtual) |
| **B2. Dynamic code translation** | Block chaining only works inside the same physical page, so the translator is called more often. The code cache is also reset on every snapshot restore |
| **B3. Syscall emulation** | The guest OS kernel and emulated hardware both run inside the emulator |

## How does it work?

The key idea is **augmented process emulation**: run the target program mostly in **user-mode QEMU** (fast), and only switch to **system-mode QEMU** (correct) when really needed.

**Workflow:**

1. Boot the firmware in system-mode emulation (using **Firmadyne**) and launch the target program. **DECAF** (virtual machine introspection) tells the fuzzer exactly when the program reaches the fork point.
2. At the fork point, walk the page table of the process and `mmap` the same physical pages (the VM's RAM is a memory-mapped file) into the user-mode emulator. Both modes now **share memory**, so migration is cheap.
3. Execute in user-mode. If there's a **page fault** (page not mapped yet), pause, pass the CPU state to system mode so the guest OS can map it, then come back.
4. If there's a **system call**, redirect it to system mode so the guest kernel handles it, then come back.
5. AFL's normal loop (seed queue, mutation, coverage bitmap) stays unchanged. Coverage is collected in user-mode QEMU like in normal AFL.

**Optimizations that matter:**

- **Preload page mappings.** Touch all code pages during bootstrap so they don't page-fault on every forked run.
- **Selective syscall redirection.** Not every syscall needs the guest kernel. The firmware's file system is mounted on the host, so file-related syscalls run locally. Only hardware or network-specific syscalls get redirected.
- **Lightweight snapshot (copy-on-write).** Mark RAM read-only, copy a page only when it's written, and on restore only write back the modified pages. This replaces QEMU's slow file-based snapshots.
- **Fork point = first network syscall**, since they target network-facing services. Input is fed by instrumenting the network syscalls directly in user-mode.

**Requirements (assumptions):** the firmware must already be emulatable in system-mode QEMU (Firmadyne), and it must run a POSIX-compatible OS (mostly Linux). Supports mipsel, mipseb, armel (90.2% of the Firmadyne images).

## Results

**Transparency (does it behave like full-system emulation?)**

- nbench ran with no errors, and 120 firmware images with HTTP services produced **identical syscall sequences** in both modes
- Proof-of-concept exploits gave identical execution traces
- Running these programs in plain user-mode QEMU: they all crashed (only `tcapi` survived)

**Overhead (standard benchmarks)**

- nbench (CPU-bound): about 0–1.5% slowdown vs user-mode
- lmbench: local syscalls are cheap, but syscalls that still need redirection are costly (e.g. `select tcp` had about 159% overhead)

**Throughput (vs TriforceAFL with lightweight snapshot as baseline)**

| Step | Speed-up |
|---|---|
| Lightweight snapshot vs stock snapshot | about 9.3× |
| + Augmented process emulation | about 3× more |
| + Selective syscall redirection | about 2.9× more |
| **Total vs best full-system setup** | **8.2× on average** |

User execution time dropped about 9× (no software address translation), and for `dnsmasq`, selective redirection alone gave another 14×. For comparison, earlier work reported under 15 test cases/s (Muench et al.) and under 1 test case/s (IoTFuzzer), while FIRM-AFL reaches hundreds per second on some programs.

**Vulnerability discovery**

- 15 known (1-day) exploits on real firmware: FIRM-AFL found the first crash **3.6× to 13× faster** than full-system emulation (e.g. 6h02 vs 21h43 for CVE-2018-19242). Setup followed Klees et al.: 10 parallel instances × 24 hours, median with confidence intervals.
- **2 zero-days** (buffer overflows in Trendnet devices): CVE-2019-11417 and CVE-2019-11418, found in about 7.5 and 6 hours. Full-system emulation found nothing in 24 hours.

## What's different from previous work?

| Previous work | Approach | Limitation | FIRM-AFL |
|---|---|---|---|
| **Avatar** | Hybrid (emulator + real hardware), whitebox | Hardware is slow, very low throughput | Pure emulation, greybox |
| **IoTFuzzer** | Blackbox via companion app, on real device | Under 1 test/s, no coverage feedback | Coverage-guided, hundreds of tests/s |
| **Muench et al.** | Blackbox fuzzing on full emulation | Up to about 15 tests/s, blind input exploration | Greybox, 8.2× faster than best full-system setup |
| **Firmadyne** | Full-system emulation for firmware | No fuzzing, and too slow for it | Uses it as the foundation, then adds speed |
| **AFL (user-mode QEMU)** | Fast greybox fuzzing | Fails on most IoT programs | Same speed, but compatible |
| **TriforceAFL** | Full-system emulation + AFL | Slow (snapshots, software MMU, syscalls) | Baseline it beats |

Main difference: it's the **first greybox fuzzer for IoT firmware**, and its contribution is a **systems-level trick** (mixing two emulation modes), not a new fuzzing algorithm. The fuzzing logic is still plain AFL.

## Can it be improved? (my thoughts)

- **Only Linux/POSIX firmware that Firmadyne can already emulate.** Many firmware images can't be emulated at all, and bare-metal or RTOS firmware (very common in small IoT devices) is out of scope. The authors admit this.
- **Only three architectures** (mipsel, mipseb, armel). They say adding more should be easy, but it's not shown.
- **Hardware is still abstracted.** Firmadyne fakes things like NVRAM, so bugs that depend on real peripherals can be missed.
- **User-space network services only.** The fork point is the first network syscall, so it fuzzes one program at a time. It doesn't cover kernel/driver bugs or multi-process interactions.
- **Network syscalls are still expensive.** The lmbench `select tcp` overhead was about 159%, so programs that do many syscalls (like `dnsmasq`) lose most of the gain unless the syscalls can be handled locally.
- **Fuzzing is not very smart.** It uses AFL with a keyword dictionary and one normal request as seed. There's no protocol or state awareness, which ChatAFL-style ideas could add (HTTP and UPnP have structure and states).
- **Crash-only oracle.** It only detects crashes, so logic bugs (which mGPTFuzz targets) are missed.
- **Most crashes were already-known bugs.** Only 2 new vulnerabilities were found, so the 0-day evidence is small.
- **Small inconsistency in the paper:** the intro says the 0-days were found within 8 hours, the evaluation says 7.5 and 6 hours, and the conclusion says "two hours". Minor, but I would double check numbers when citing.
- **Baseline fairness.** TriforceAFL had to be modified by the authors (snapshot support and VMI) to be a fair baseline. It's a reasonable choice, but it's still a comparison against their own re-implementation.

## Comparison with mGPTFuzz and ChatAFL

Now that I have read three papers, here is how they fit together:

| | FIRM-AFL | ChatAFL | mGPTFuzz |
|---|---|---|---|
| Main contribution | **Systems**: fast emulation | **Input generation**: LLM-guided grammar, seeds, states | **Oracle**: spec-based FSMs for logic bugs |
| Fuzzing type | Greybox (emulated, instrumented) | Greybox (source instrumented) | Blackbox (real device) |
| Target | Linux IoT firmware (HTTP/UPnP/CGI services) | Open-source network servers | Matter IoT devices |
| Needs firmware? | Yes (and it must be emulatable) | Needs source code to instrument | No (talks to the real device) |
| Knows protocol/state? | No | Yes (LLM) | Yes (LLM + spec) |
| Bug types | Crashes | Crashes (ASAN) | Crashes + non-crash logic bugs |
| Fuzzing algorithm | Plain AFL | AFLNet + LLM | Rule-based policies over FSMs |

They solve different layers of the same problem:

- **FIRM-AFL** answers: *how do I run the target fast enough to fuzz it?*
- **ChatAFL** answers: *what inputs should I send, in what order?*
- **mGPTFuzz** answers: *how do I know the device behaved wrongly if it didn't crash?*

mGPTFuzz also explains why it avoids emulation (it mentions that emulating IoT firmware is hard and that many vendors don't release firmware), which is exactly the limitation FIRM-AFL runs into.

## Conclusion / what I learned

FIRM-AFL shows that **throughput is a first-class concern in fuzzing**, and that you can sometimes win big with systems engineering instead of a smarter algorithm. Key takeaways:

- **Greybox fuzzing needs speed.** The paper's whole premise is that executions per second drive coverage, so removing overhead (SoftMMU, code translation, syscalls) is a valid research contribution.
- **Profile before optimizing.** They broke the cost into B1, B2, B3 and then attacked each one, and also showed a time breakdown for each optimization.
- **Best of both worlds via a hybrid design.** Run the common case on the fast path (user-mode) and fall back to the slow-but-correct path (system-mode) only when needed. This is similar to a fast-path/slow-path design in other systems.
- **Snapshots matter.** The copy-on-write lightweight snapshot gave a bigger boost (9.3×) than the headline technique alone, which shows that boring engineering details can matter more than the main idea.
- **Evaluation done right.** They used 10 parallel runs × 24 hours and confidence intervals, following *Evaluating Fuzz Testing* (Klees et al.), which I want to read next.
- **Emulation is a bottleneck for IoT fuzzing in general.** It limits which devices you can test, which explains why blackbox approaches (IoTFuzzer, Snipuzz, mGPTFuzz) exist.

**For my own fuzzing learning journey:** this paper gave me the "infrastructure" part of the picture. Before, I mostly thought about fuzzing in terms of mutation and coverage. Now I also see the target execution environment (emulation, snapshots, fork servers) as something to design. I also understand better why AFL's fork server and QEMU mode matter, and how ChatAFL's improvements sit on top of that.

**Next steps:**

- [x] Read FIRM-AFL (USENIX Security 2019) for the greybox/emulation side of IoT fuzzing
- [ ] Read *Evaluating Fuzz Testing* (Klees et al., CCS 2018) to understand the evaluation method used here
- [ ] Read *What You Corrupt Is Not What You Crash* (Muench et al., NDSS 2018) on challenges of fuzzing embedded devices
- [ ] Read the Firmadyne paper (NDSS 2016), since FIRM-AFL depends on it
- [ ] Learn how AFL's fork server and QEMU mode work (the original AFL docs)
- [ ] Think about combining ideas: fast emulation (FIRM-AFL) + structure/state guidance (ChatAFL) + spec oracle (mGPTFuzz)
