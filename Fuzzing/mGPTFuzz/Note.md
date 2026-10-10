# Paper Summary: mGPTFuzz

**Paper:** *From One Thousand Pages of Specification to Unveiling Hidden Bugs: Large Language Model Assisted Fuzzing of Matter IoT Devices*
**Authors:** Xiaoyue Ma, Lannan Luo, Qiang Zeng (George Mason University)
**Venue:** USENIX Security 2024
**Link:** https://www.usenix.org/conference/usenixsecurity24/presentation/ma-xiaoyue
**Tool page:** https://iot-fuzz.github.io

## Table of Contents

- [What problem does it solve?](#what-problem-does-it-solve)
- [How does it work?](#how-does-it-work)
- [Results](#results)
- [What's different from previous work?](#whats-different-from-previous-work)
- [Can it be improved?](#can-it-be-improved-my-thoughts)
- [Conclusion / what I learned](#conclusion--what-i-learned)

---

## What problem does it solve?

Matter is a new IoT standard backed by 200+ companies (Apple, Google, Amazon, Samsung), so a huge number of smart home devices now speak it. Nobody had built a fuzzer for it. The authors point out four things that make this hard:

1. **The spec is huge.** It runs to 1,258 pages, and turning that into something a fuzzer can use by hand is slow and error-prone.
2. **Stateful bugs.** Many commands only make sense in certain states (e.g. "turn off" only matters if the light is on), so a bug may only trigger from one specific state.
3. **Non-crash bugs.** You can't instrument IoT firmware (code signing), so you can't use coverage, and logic errors don't crash the device, which means there's no obvious signal that anything went wrong.
4. **Command coverage.** Knowing every command a device supports normally means reading manuals or reverse engineering the companion app.

## How does it work?

The overall idea is **blackbox, specification-guided fuzzing, where an LLM reads the spec for you.**

- **Custom controller as the fuzzer.** Matter devices verify the controller's certificate much less strictly than they verify device certificates. So the authors built a custom controller (modifying the Matter Consortium's `chip-tool`) and used it to commission and fuzz devices. This also gets around encryption, because messages are generated in plaintext and then encrypted by the normal stack.
- **Full command coverage for free.** When a device is paired, it announces all its clusters, commands and attributes. The fuzzer reads those pairing messages, so there's no need for API test scripts or manuals.
- **LLM builds a knowledge base.** GPT-4 reads the spec cluster by cluster and extracts commands, attributes, data types, value ranges and IDs. It then builds a **finite state machine (FSM)** for each cluster, where nodes are device states and edges are commands. The result was 52 FSMs, 521 states and 522 transitions, generated in about 15 minutes.
- **Keeping the LLM reliable.** They used temperature 0, few-shot examples, chain-of-thought prompts, self-consistency (majority vote over multiple runs), and explicit "don't make stuff up" instructions. They also checked the output manually: 100 sampled transitions were all correct, and one cluster was fully checked for coverage.
- **Fuzzing policies.**
  - Policy 1: valid, boundary and random values, plus long, empty and NULL inputs
  - Policy 2: change argument types
  - Policy 3: change the number of arguments
  - Policy 4: try unsupported clusters and commands
- **Removing input sanitization.** The controller's APIs normally reject invalid inputs, so they stripped the sanitization out of the message packing procedures. This is a one-time effort rather than a per-device one.
- **Detecting bugs with the spec as an oracle.** Crashes show up as disconnects or timeouts. For non-crash bugs, they check whether the response and the resulting attribute values match what the FSM says should happen. A valid command should return `SUCCESS` and the right state, and an invalid one should return an error like `INVALID_COMMAND`.

## Results

- 23 real consumer devices (Kasa, Philips Hue, Yale, Govee and others)
- **147 new bugs**: 5 crash bugs and 142 non-crash bugs, with 10 stateful ones
- 61 count as vulnerabilities (DoS) and 3 CVEs were assigned (CVE-2023-42189, CVE-2023-45955, CVE-2023-45956)
- SNIPUZZ found **zero** bugs, even after 24 hours per device
- One bug (CVE-2023-42189) affects *all* Matter devices. Removing key set ID 0 (the IPK) was wrongly allowed, and the spec says it must fail. The developers had missed that sentence.

## What's different from previous work?

| Previous work | Limitation | mGPTFuzz |
|---|---|---|
| IoTFuzzer, DIANE | Reverse engineer the companion app per device, which breaks with obfuscation, and Matter devices usually can't be controlled by the vendor app anyway | Controller-based, so no app analysis |
| SNIPUZZ | Needs vendor API scripts (only 6 of 23 devices had them), covers few commands, and breaks on encrypted traffic | Gets commands from pairing messages and works on plaintext |
| HubFuzzer | Same hub idea, but only ZigBee and Z-Wave | Applies the idea to Matter |
| ChatAFL, Fuzz4All and other LLM fuzzers | Greybox (need instrumentation) and mostly find crashes | First LLM-assisted **blackbox** fuzzer, finds non-crash and stateful bugs |
| BrakTooth | Infers state machines from packets, at the Bluetooth link layer | Extracts FSMs from the spec at the application layer |

The main differences are that the spec itself acts as the test oracle, which is how it finds logic bugs without any instrumentation, and that state is handled systematically by walking FSMs.

## Can it be improved? (my thoughts)

- **It only fuzzes Matter.** The authors say the approach should generalise to Zigbee, Thread and Bluetooth, but that isn't shown.
- **The LLM validation is thin.** 100 of 522 transitions plus one full cluster is decent, but a wrong FSM would produce false positives or miss real bugs. There's also no measurement of the false-positive rate for non-crash bugs. Some "bugs" might be spec ambiguities or the LLM misreading a table (they also used OCR on the PDF, which is another error source).
- **Bug counting is a bit generous.** 142 of the 147 bugs are non-crash, and many look like the same root cause across devices (e.g. the SDK bug). The bug counts also aren't clearly deduplicated, and "61 zero-day vulnerabilities" depends on classifying DoS as a vulnerability.
- **Baseline comparison is weak.** SNIPUZZ is not built for non-crash bugs and had to be bent to fit, so zero vs 147 isn't a very fair fight. A better baseline would be something like a spec-driven fuzzer with hand-written grammars, or an LLM-free ablation of mGPTFuzz.
- **No ablation.** It never shows how much the FSMs help compared with the policies alone, or how much the stateful part matters (only 10 bugs were stateful).
- **Fuzzing is blind within the policies.** There's no feedback loop (coverage isn't available), so it's a fixed set of mutation rules and not adaptive. Using the device's responses to guide mutation could find more.
- **Dependence on a closed model.** GPT-4 is a black box that changes over time, so reproducibility is a concern. An open model could be tested.
- **Only the application layer.** It doesn't test commissioning, the transport layer or radio-level issues.

## Conclusion / what I learned

This paper shows that when you can't instrument the target, you can still get a strong signal by using **the spec as an oracle.** The key ideas to take away:

- Blackbox fuzzing doesn't have to mean "random mutation + watch for crashes." Semantic knowledge of what *should* happen lets you detect logic bugs.
- Stateful fuzzing needs an explicit model of state. Here an FSM was used to drive test generation instead of treating every input as independent.
- LLMs are useful as a tool for turning unstructured documents into structured data, as long as you add guardrails (low temperature, few-shot, self-consistency, human validation).
- Architecture choices can remove whole problems. Fuzzing from the controller side sidestepped encryption, app reverse engineering and command discovery in one go.
- Bugs often come from developers missing one sentence in a huge spec, so spec-vs-implementation checking is a good fuzzing target for any protocol with a long standard.

**For my own learning journey:** it ties together protocol fuzzing, stateful fuzzing and LLM-assisted fuzzing, and it made me realise that how you build your oracle matters as much as how you generate inputs.

**Next steps:**

- [ ] Read ChatAFL (NDSS 2024) to compare greybox LLM fuzzing with this blackbox approach
- [ ] Read Snipuzz (CCS 2021) to understand the baseline properly
- [ ] Think about how an FSM + oracle design could be applied to other protocols
