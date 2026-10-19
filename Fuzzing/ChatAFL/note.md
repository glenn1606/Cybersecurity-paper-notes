# Paper Summary: ChatAFL

**Paper:** *Large Language Model guided Protocol Fuzzing*
**Authors:** Ruijie Meng, Martin Mirchev, Marcel Böhme, Abhik Roychoudhury (NUS, MPI-SP, Monash)
**Venue:** NDSS 2024
**Code:** https://github.com/ChatAFLndss/ChatAFL

## Table of Contents

- [What problem does it solve?](#what-problem-does-it-solve)
- [How does it work?](#how-does-it-work)
- [Results](#results)
- [What's different from previous work?](#whats-different-from-previous-work)
- [Can it be improved?](#can-it-be-improved-my-thoughts)
- [Comparison with mGPTFuzz](#comparison-with-mgptfuzz)
- [Conclusion / what I learned](#conclusion--what-i-learned)

---

## What problem does it solve?

Protocol implementations (FTP, RTSP, SIP, SMTP...) face the internet, so a bug in them is very serious (often remote code execution). They are also **stateful**: you need the right message, in the right format, in the right order. The protocol rules are written in RFCs, which are hundreds of pages of natural language, so a fuzzer can't read them directly.

The current state of the art (AFLNet, NSFuzz) are **mutation-based**: they record real message sequences as seeds and mutate them. The authors list three weaknesses:

1. **C1: Dependence on initial seeds.** The recorded seeds don't cover all protocol states and message types. For Live555 (RTSP), the seeds only had 4 out of 10 request types.
2. **C2: Unknown message structure.** The fuzzer doesn't know the format, so mutations often just break the message (it gets rejected) and don't explore anything deeper.
3. **C3: Unknown state space.** The fuzzer can't tell which message would move the server into a new state, so it gets stuck on a coverage plateau.

## How does it work?

The idea is **LLM-guided protocol fuzzing**: ChatGPT has already read the RFCs during training, so just ask it. ChatAFL is built on top of AFLNet and adds three LLM-based strategies:

| Strategy | What the LLM does | Fixes |
|---|---|---|
| **SA: Grammar-guided mutation** | Asked once, before fuzzing, for the grammar of every client request type. Fixed keywords stay as they are and mutable parts become `<Value>`. The fuzzer then only mutates the `<Value>` parts (with some probability, so classic random mutation is still used for corner cases) | C2 |
| **SB: Enriching initial seeds** | Looks at the existing seed sequence, finds which request types are missing and inserts them in the right place (two missing types at a time to keep seeds short) | C1 |
| **SC: Surpassing coverage plateau** | When the fuzzer produces 512 uninteresting seeds in a row (`MaxPlateau`), the LLM is given the message history and asked "what is the next client request that would change the server state?" | C3 |

**Prompt engineering details worth remembering:**

- **Few-shot with two examples** from different protocols (RTSP/HTTP), so the LLM doesn't copy one example too closely
- **Self-consistency**: ask 5 times and take the majority answer, to filter out random junk (e.g. a made-up `SET_DESCRIPTION` request)
- **Temperature**: 0.5 for grammar and seeds (want accuracy), 1.5 for plateau breaking (want creativity)
- **Continuation-style prompts** ("the modified sequence is:") so the output can be used directly as a seed
- Include real server responses in the prompt so the LLM gets context like **session IDs**
- Model: `gpt-3.5-turbo`
- The LLM is only used at start-up and at plateaus, which keeps the cost down. The rest is normal AFLNet speed.

**Case study that motivated the design (Live555 / RTSP):**

- Grammar: 9 of 10 request types matched the ground truth (2 authors spent 8 hours building it). PLAY missed the optional `Range` field in 15 of 50 answers.
- Seed enrichment: 99% of the 100 generated requests were placed correctly. About 55% were accepted by the server, and the failures were mostly unsupported features and wrong session IDs.
- State transitions: 69–89% of the LLM's suggested next messages caused a real state change.

## Results

Evaluated on 6 text-based protocol servers from ProFuzzBench (Live555, ProFTPD, PureFTPD, Kamailio, Exim, forked-daapd), 10 runs of 24 hours each.

| Metric | vs AFLNet | vs NSFuzz |
|---|---|---|
| State transitions | **+47.60%** (48× faster to same coverage) | +42.69% (16× faster) |
| States | +29.55% | +25.75% |
| Branches (code) | +5.81% (6× faster) | +6.74% (10× faster) |

- **Ablation (branch coverage):** SA alone gives +3.04%, SA+SB gives +3.86%, all three give +5.81%. Every strategy helps and none hurts.
- **9 zero-day bugs** found (7 in Live555, 1 in ProFTPD, 1 in Kamailio). 7 are potentially security-critical (use-after-free, heap overflow). AFLNet found only 3 of them and NSFuzz found 4. 7 were confirmed by developers and 3 fixed.
- **Example bug #1:** a use-after-free that needs the sequence `SETUP → PLAY → PAUSE → PLAY`. None of the baseline runs ever generated a `PAUSE` message, but ChatAFL's seed enrichment added it. This shows all three strategies working together.
- **Manual effort:** only about 1 hour to write the two few-shot examples, once. After that it works on any protocol by changing the protocol name.

## What's different from previous work?

| Previous work | Limitation | ChatAFL |
|---|---|---|
| Generation-based (Peach, boofuzz) | Needs hand-written data and state models. Tedious and incomplete | LLM provides the spec info automatically |
| Dynamic message inference (TreeFuzz, Polyglot) | Can only infer structure from messages already observed | LLM knows structures that never appeared in the seeds |
| Mutation-based (AFLNet, NSFuzz, StateAFL, SGFuzz) | Seed-limited, structure-unaware, state defined by response codes or program variables | Keeps AFLNet's feedback loop but adds structure-awareness and state guidance |
| LLM fuzzers (CodaMosa, TitanFuzz, FuzzGPT, ChatFuzz) | Mostly generate or mutate inputs by querying the LLM constantly | **Separates info extraction from fuzzing**: extract grammar once, fuzz fast, and only call the LLM when stuck |

Main differences: it's the first LLM-guided *stateful* protocol fuzzer, and it doesn't try to define what a state is. It delegates that to the LLM.

## Can it be improved? (my thoughts)

- **Text-based protocols with public RFCs only.** Binary protocols and proprietary ones (whose docs aren't in the training data) may not work well. The authors admit this.
- **It relies on the LLM's memory, not on the actual spec.** The LLM can hallucinate or forget optional fields (the missing `Range` case). Feeding the real RFC text into the prompt (like mGPTFuzz does) could be more reliable.
- **State is still defined by response codes** (AFLNet style). That's a coarse approximation, and the paper has no ground-truth state machine to check against.
- **Coverage gains in code are modest** (about 6%). The big wins are in states and transitions, and coverage is only a proxy for finding bugs.
- **Bug results are concentrated in Live555** (7 of 9). It's not clear how well this generalises, and 9 bugs from 6 targets is a small sample.
- **Case study is on a single protocol** (RTSP), so the design decisions might be tuned for it.
- **Ablation is cumulative** (SA, then SA+SB, then all). It doesn't test each strategy alone, so the individual contributions are not fully separated.
- **Closed model and cost.** `gpt-3.5-turbo` can change over time (reproducibility), and the temperature values were picked heuristically. An open model or a fine-tuned one might do better or cheaper.
- **No oracle for logic bugs.** It still relies on ASAN crashes, so non-crash bugs are out of scope.
- **No comparison with generator-based fuzzers** that have good hand-written specs. That would show how much the LLM really replaces manual work.

## Comparison with mGPTFuzz

Since I read mGPTFuzz first, here is how the two compare:

| | ChatAFL | mGPTFuzz |
|---|---|---|
| Fuzzing type | Greybox (instrumented, uses code and state coverage) | Blackbox (no instrumentation) |
| Target | Text-based internet protocols (RTSP, FTP, SIP...) | Matter IoT devices |
| Where spec knowledge comes from | LLM's pre-trained memory of RFCs | Spec text fed to the LLM in the prompt |
| Knowledge format | Grammar of message types | FSMs (states, commands, value ranges) |
| When LLM is used | During fuzzing (seed enrichment, plateau) plus a one-time grammar | Offline, one time, to build the knowledge base |
| What guides fuzzing | Coverage feedback | The spec as an oracle |
| Bug types | Crashes (memory errors via ASAN) | Crashes plus non-crash logic bugs |

Both papers use the same prompt tricks (few-shot, self-consistency, low temperature), and both say "LLM turns human-readable spec into machine-readable info".

## Conclusion / what I learned

ChatAFL shows that an LLM can fill the gap that a mutation-based protocol fuzzer has: it doesn't know the message format, the state machine or which messages are missing. The key lessons:

- **Use the LLM where the fuzzer is weak, not everywhere.** The LLM is called once for grammar and seeds, and then only on plateaus. This keeps the speed of a normal fuzzer.
- **The three problems (seeds, structure, state) map to three small, separate LLM strategies.** That makes it easy to ablate and understand.
- **Valid-but-weird inputs matter.** Structure-aware mutation keeps messages valid, so they get past the parser and reach deeper code and states.
- **Stateful bugs need the right sequence.** Bug #1 needed 4 specific messages in order, which random mutation almost never gets.
- **Prompt engineering is part of the fuzzer design**: few-shot, majority vote, temperature choice and the output format all affect whether the fuzzer can parse the answer.
- **Coverage is a proxy.** It's good that they backed it up with real zero-day bugs.

**For my own fuzzing learning journey:** this paper (together with mGPTFuzz) gave me a clear picture of two ways to bring LLMs into fuzzing. One is greybox with LLM in the loop (ChatAFL) and the other is blackbox with LLM as a spec reader (mGPTFuzz). It also made me want to understand AFLNet properly, since ChatAFL is basically "AFLNet plus three LLM hooks".

**Next steps:**

- [x] Read ChatAFL (NDSS 2024) to compare greybox LLM fuzzing with the blackbox approach
- [ ] Read AFLNet (ICST 2020) to understand the baseline and its state model
- [ ] Read NSFuzz and StateAFL to compare how they define "state"
- [ ] Run the ChatAFL artifact on a small target (e.g. `pure-ftpd` for 5 minutes, as in the artifact appendix)
- [ ] Think about combining both ideas: spec-fed FSMs plus coverage feedback
