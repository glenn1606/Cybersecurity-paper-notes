# FlowFence: Practical Data Protection for Emerging IoT Frameworks

> **Original Paper:** Fernandes et al., *"FlowFence: Practical Data Protection for Emerging IoT Application Frameworks"*, 25th USENIX Security Symposium (2016).  
> **Source Paper Link:** [USENIX Security '16 PDF](https://www.usenix.org/system/files/conference/usenixsecurity16/sec16_paper_fernandes.pdf)

---

## 📌 Overview & Motivation

### Why FlowFence?
Modern IoT application frameworks (e.g., Samsung SmartThings, Apple HomeKit, Google Fit) rely on **permission-based access control**. While permissions act as gatekeepers to sensitive data sources (cameras, door locks, heart rate sensors), they offer **zero control** over how applications process or transmit that data once access is granted.

This introduces severe security risks:
* An app granted access to a camera (to unlock a door via face recognition) and the Internet (to send lock state notifications) can secretly exfiltrate raw camera streams to the web.
* Existing solutions like Dynamic Taint Analysis incur massive performance/memory overhead and struggle with implicit flows or concurrency.

### What is FlowFence?
**FlowFence** is an IoT security architecture that enforces **Information Flow Control (IFC)** between data sources (sensors/publishers) and data sinks (actuators/network). It forces third-party apps to declare their data flow policies at installation and guarantees that **undeclared flows are strictly blocked at runtime**.

---

##  How It Works: Opacified Computation

FlowFence introduces **Opacified Computation**, a model where apps are split into two distinct execution tiers:
+-----------------------------------------------------------------------+
|                         Non-Sensitive Code                            |
|             (Orchestrates execution, handles handles only)             |
+-----------------------------------------------------------------------+
|                                                       ^
| Invokes QM with Opaque Handles                        | Returns
v                                                       | Opaque Handle
+-----------------------------------------------------------------------+
|                    FlowFence Trusted Service                          |
|  - Tracks Taints    - Enforces Policies    - Dereferences Handles     |
+-----------------------------------------------------------------------+
|                                                       ^
| Executes in Sandboxed Environment                     | Raw Output
v                                                       |
+-----------------------------------------------------------------------+
|                    Quarantined Modules (QMs)                          |
|                (Process sensitive raw data in isolation)             |
+-----------------------------------------------------------------------+

```markdown
# FlowFence: Practical Data Protection for Emerging IoT Frameworks

> **Original Paper:** Fernandes et al., *"FlowFence: Practical Data Protection for Emerging IoT Application Frameworks"*, 25th USENIX Security Symposium (2016).  
> **Source Paper Link:** [USENIX Security '16 PDF](https://www.usenix.org/system/files/conference/usenixsecurity16/sec16_paper_fernandes.pdf)

---

## 📌 Overview & Motivation

### Why FlowFence?
Modern IoT application frameworks (e.g., Samsung SmartThings, Apple HomeKit, Google Fit) rely on **permission-based access control**. While permissions act as gatekeepers to sensitive data sources (cameras, door locks, heart rate sensors), they offer **zero control** over how applications process or transmit that data once access is granted.

This introduces severe security risks:
* An app granted access to a camera (to unlock a door via face recognition) and the Internet (to send lock state notifications) can secretly exfiltrate raw camera streams to the web.
* Existing solutions like Dynamic Taint Analysis incur massive performance/memory overhead and struggle with implicit flows or concurrency.

### What is FlowFence?
**FlowFence** is an IoT security architecture that enforces **Information Flow Control (IFC)** between data sources (sensors/publishers) and data sinks (actuators/network). It forces third-party apps to declare their data flow policies at installation and guarantees that **undeclared flows are strictly blocked at runtime**.

---

## 🏗️ How It Works: Opacified Computation

FlowFence introduces **Opacified Computation**, a model where apps are split into two distinct execution tiers:


```

+-----------------------------------------------------------------------+
|                         Non-Sensitive Code                            |
|             (Orchestrates execution, handles handles only)             |
+-----------------------------------------------------------------------+
|                                                       ^
| Invokes QM with Opaque Handles                        | Returns
v                                                       | Opaque Handle
+-----------------------------------------------------------------------+
|                    FlowFence Trusted Service                          |
|  - Tracks Taints    - Enforces Policies    - Dereferences Handles     |
+-----------------------------------------------------------------------+
|                                                       ^
| Executes in Sandboxed Environment                     | Raw Output
v                                                       |
+-----------------------------------------------------------------------+
|                    Quarantined Modules (QMs)                          |
|                (Process sensitive raw data in isolation)             |
+-----------------------------------------------------------------------+

```

### 1. Quarantined Modules (QMs)
* Small, developer-written functions that compute on sensitive data inside isolated, system-provided sandboxes (Android `isolatedProcess`).
* Treated as **black boxes** by the framework: input arguments are automatically dereferenced into raw data, and outputs are intercepted.

### 2. Opaque Handles
* When a QM finishes execution, the result is converted into an **Opaque Handle** before returning to the app's orchestration code.
* Opaque Handles are **immutable, references to sensitive data** bound to a **taint set** tracking provenance.
* Outside a sandbox, non-sensitive code cannot read, inspect, or branch on handle values, effectively **preventing implicit information flows**.

### 3. Dynamic Taint Tracking & Policy Enforcement
* **Taint Accumulation:** A sandbox inherits the combined taint set of all data handles it dereferences.
* **Transitive Flow:** Any output produced by a sandbox retains all accumulated taints.
* **Declassification via Trusted APIs:** Exporting data to a sink (network, lock, screen) requires calling a FlowFence `TrustedAPI`. The framework matches the sandbox's accumulated taint set against declared `<source, sink>` policies. If undeclared, the operation is blocked immediately.

---

##  Highlights, Advantages & Real Benchmarks

* **Zero Custom OS/Hardware:** Built using standard OS primitives (process isolation & IPC) on Android (Nexus 4 used as an IoT Hub).
* **Low Memory Footprint:** Core service takes **6.35 MB RAM**; each sandbox consumes ~**2.7 MB RAM** (16 active sandboxes take only ~49.5 MB).
* **Minimal Latency Overhead:** Inter-sandbox execution latency is **$\le$ 92 ms**.
* **Real-World App Validation:**
  * **FaceDoor** (Face recognition door lock): **+4.9%** latency overhead (3117ms $\rightarrow$ 3270ms).
  * **HeartRateMonitor** (Camera-based PPG measuring): Negligible throughput loss (< 1% bandwidth reduction).
  * **SmartLights** (Location-triggered lighting): Successfully prevented background GPS exfiltration.
* **Low Porting Effort:** Average app size increased from ~232 to ~332 lines of code. Porting 3 real-world apps took a single developer **5 days**.

---

## ⚠️Limitations & Future Work

* **Side-Channel Leaks:** QM execution time can potentially encode sensitive bits (timing attacks). Mitigation requires scheduling QMs using deterministic or predictive timing bounds.
* **Overtainting & Poison-Pill Attacks:** Malicious publishers could broadcast heavily-tainted data onto public channels to intentionally "poison" consumers and prevent them from sinking data. FlowFence mitigates this via **Taint Bounds ($TM_c$)** on channels.
* **Usability of Flow Prompts:** Users may blindly approve flow policies during installation. Future iterations can incorporate flow frequency metrics (e.g., *"App accessed GPS 5,000 times today"*) to prompt security adjustments.

---

##  Key Takeaways for Custom Implementation

If you plan to implement an Opacified Computation model yourself:

1. **Keep Sandboxes Ephemeral/Reusable:** Reusing sandboxes without wiping memory is safe **only if** the new QM's input taint set is a **superset** of the sandbox's current taint set ($\text{Taint}_{\text{new}} \supseteq \text{Taint}_{\text{current}}$).
2. **Immutable Handles are Essential:** Never allow in-place modification of Opaque Handles to prevent cross-module overtainting.
3. **Pass-by-Value KV Store:** When building intermediate state storage (Key-Value stores or Event Channels), ensure all reads clone data values so taint sets remain isolated across time.
4. **Development Debug Flag:** Provide a debug mode that temporarily lifts handle opaqueness so stack traces and data types are visible during development.

```