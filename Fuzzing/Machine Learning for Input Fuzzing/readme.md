# Learn&Fuzz: Machine Learning for Input Fuzzing — Notes

**Authors:** Patrice Godefroid, Hila Peleg, Rishabh Singh (Microsoft Research / Technion)
**Paper:** [arXiv:1701.07232](https://arxiv.org/abs/1701.07232)

Notes covering the **Introduction** and **Section 2 (Structure of PDF Documents)**.

---

## 1. Introduction

### The problem
Fuzzing means repeatedly feeding a program modified ("fuzzed") inputs to try to trigger crashes or security bugs in the code that parses those inputs. The paper identifies three standard fuzzing approaches:

1. **Blackbox random fuzzing** — fully automatic, mutate bytes randomly.
2. **Whitebox constraint-based fuzzing** — fully automatic, uses symbolic execution/constraints.
3. **Grammar-based fuzzing** — generates inputs from a grammar describing the input format; historically the *most effective* method for complex formats (e.g. browsers parsing HTML/JS), but it is **not automatic** — someone has to hand-write the grammar, which is slow and error-prone.

### The core idea
The paper proposes **automatically learning an input grammar** from a corpus of sample inputs, using neural-network-based statistical machine learning (rather than classic automata/CFG-learning techniques used in prior work). Specifically, they use **recurrent neural networks (RNNs)** to learn a statistical, *generative* model of the input format — one that can both describe and generate new inputs by sampling from the learned probability distribution.

- Learning is **unsupervised** and **fully automatic** — no format-specific hand tuning required.

### Case study
They apply this to a genuinely hard case: the **PDF format**, whose official spec runs to ~1,300 pages, and target the **PDF parser embedded in Microsoft Edge**.

### The central tension ("learn&fuzz")
A key theme introduced here (and explored throughout the paper): **learning** wants to faithfully capture the structure of *well-formed* inputs, while **fuzzing** wants to deliberately *break* that structure to reach unusual/unexpected code paths (error handlers, edge cases) where bugs hide. These two goals pull in opposite directions, and balancing them is the "learn&fuzz" challenge.

### Contributions
- First use of neural-network (seq2seq) statistical learning for grammar-based fuzzing.
- A detailed case study learning PDF object structure from real PDFs and fuzzing the Edge PDF parser.
- A new algorithm, **SampleFuzz**, which uses the learned probability distribution itself to decide *where* to intelligently inject fuzzing perturbations, and which is shown to outperform the other learning-based and random baselines tested.

### Paper roadmap
- §2 — Overview of the PDF format and the scope of the work
- §3 — Neural-network learning background and their learn&fuzz approach
- §4 — Experiments on the Edge PDF parser
- §5 — Related work
- §6 — Conclusion / future work

---

## 2. The Structure of PDF Documents

### Overview
PDF's full spec is 1,300+ pages, and roughly **70% of it** concerns *data objects* and how they relate to one another — this is exactly the part the paper targets for learning.

A PDF file is stored as **text** (which may embed binary streams, e.g. images or encrypted data). A PDF **document** = a sequence of one or more PDF **bodies**. Each body has three parts:

1. **Objects**
2. **Cross-reference table**
3. **Trailer**

### 2.1 Objects
The basic unit of data/metadata in a PDF. All objects share a common outer structure:

- First line: `<id> <generation-number> obj` — the id/generation number let the object be referenced elsewhere (the generation number increments if the object is later overwritten).
- Closed by `endobj`.
- Can contain a **dictionary** (delimited by `<<` ... `>>`) with `/key value` pairs.
- Objects can reference each other, e.g. `[3 0 R]` means "indirect reference to object id 3, generation 0."
- Because files can be large, references are resolved via random access using the **cross-reference table** rather than a linear scan.

**Other object types shown in the paper's examples:**
- **Array**: e.g. `[680.6 680.6]` — holds a list of values (like coordinates).
- **String literal**: e.g. `(Related Work)` — holds text such as a bookmark label.
- **Numeric object**: e.g. `4171`.
- **Mixed-type array**: e.g. `[false 1708 5.5 (Hello) /My#20Name]` — combines booleans, numbers, strings, and names in one array.

These object types are both used standalone and as building blocks composed into larger objects (e.g. a dictionary object containing an array). The rules governing how objects are defined and composed make up most of the PDF spec.

### 2.2 Cross-reference table
Stores the **byte address** of every object in the document so objects can be randomly accessed instead of parsed linearly.

- Organized in subsections; each row corresponds to one object ID.
- `n` = object is in use (row gives its byte address in the file).
- `f` = object is free/unused (row instead links to the ID of the *next* free object, forming a free-list; object 0 always points to placeholder ID `65535`, closing the chain).

### 2.3 Trailer
Contains:
- A dictionary with document-level metadata (e.g. `/Root`, `/Info`, `/Size`).
- `startxref` — the byte address of the cross-reference table.

This design lets a PDF reader **parse from the end of the file**: read `startxref` → jump to the cross-reference table → parse it → only parse individual objects lazily, as needed.

### 2.4 Incremental updates
PDFs support **incremental updates**: to "edit" an existing object (say, object 12), a writer doesn't rewrite the file — it appends a **new body** containing a new version of object 12 with an incremented generation number, followed by a new cross-reference table pointing to it, which then gets appended to the original file. Deletion works similarly, by marking an object free in a new cross-reference table. The paper reuses this exact mechanism later (§4) to append their machine-generated objects into real "host" PDF files for testing.

### 2.5 Scope of this work (what they do and don't try to learn)
The authors deliberately narrow their target:

 **In scope:** learning the grammar of **non-binary PDF data objects** (formatted text objects like those above) — this is the bulk (~70%) of the spec, and it's **repetitive and structured**, which suits neural-network learning well.

 **Out of scope:**
- **Cross-reference tables and trailers** — these involve numeric/pointer/counter constraints (addresses, counts) that seemed too complex and less promising to learn via neural nets.
- **Binary data objects** (e.g. embedded images) — existing blackbox/whitebox fuzzing techniques already handle these effectively, so there was no need to target them with this approach.

---

*(Notes end at Section 2; Sections 3 onward cover the seq2seq model details, the SampleFuzz algorithm, and the experimental results on the Edge PDF parser.)*

# 1. Overview of Seq2Seq (Sequence-to-Sequence) Model

- **Architecture (2 RNN Networks):**
  - **Encoder RNN:** Compresses a variable-length input sequence into a fixed-dimensional context representation.
  - **Decoder RNN:** Takes the fixed-dimensional context representation from the Encoder and generates a variable-length output sequence.
- **Generation Mechanism:** The predicted character or token at time step `t` is fed directly as the input for the next time step `t + 1`.
- **Conditional Probability:** Learns the conditional probability distribution `p(y_1, ..., y_T1 | x_1, ..., x_T2)` to generate target sequence `y` given input sequence `x`.

---

# 2. Chapter 3: Statistical Learning of Object Contents

- **Objective:** Learns a generative language model over the character sequences of PDF objects.
- **Key Advantage:** Overcomes the limited context window of traditional `n`-gram models by leveraging the seq2seq architecture.
- **New Data Generation:** Once trained, the model generates new PDF objects by sampling from the learned probability distribution starting with an initial prefix (e.g., `"obj"`).

---

# 3. Data Preparation & Training Pipeline (Section 3.1)

- **PDFs as Character Sequences:** Treats all PDF object files purely as raw sequence strings.
- **Data Concatenation:** Concatenates all individual object files `s_1, s_2, ..., s_n` into one large unified character sequence `s_tilde`.
- **Training Samples Splitting:** Slices the continuous sequence `s_tilde` into multiple fixed-size input chunks `t_i` of length `d`.
- **Target Output Sequence Creation:** The target output sequence `o_t` is created by shifting the input sequence to the right by 1 character position (enabling the model to predict the next character).
- **End-to-End Training:** Trains the seq2seq model end-to-end directly on these paired instances to learn the generative language model.


**SAMPLE FUZZ ALGORITHM**
Step-by-Step Breakdown
Start with a Prefix

The algorithm begins building a text string starting with "obj ".

Loop Until Finished

It keeps predicting and appending characters one by one until it generates "endobj" (or hits a maximum safety length MAXLEN).

Predict the Next Character

At each step, it samples a candidate character c and gets its predicted probability p(c) from the trained model.

Decide Whether to Fuzz

The algorithm rolls a random probability p_fuzz.

Fuzzing Trigger: Fuzzing occurs only if two conditions are met:

p_fuzz > t_fuzz (a threshold condition to keep fuzzing rare and controlled).

p(c) > p_t (the AI is highly confident that c is the correct character).

The Swap: Instead of using the confident character c, it replaces c with c', the character with the lowest likelihood in the probability distribution.

Append & Safety Check

It appends the selected character (c or fuzzed c') to the sequence. If the sequence gets too long without finishing, it resets back to "obj " to prevent infinite loops.