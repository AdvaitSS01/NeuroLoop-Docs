# Best Practices for Debugging and Validation in Closed-Loop Experiments

## 1. Concept definition

Closed-loop experiments operate on the knife-edge between **control** and **uncertainty**.  
Every trigger, frame, and TTL pulse must not only occur — it must be *verifiable*.  
Debugging in such systems is not about fixing code alone, but about **tracing causal failure through time**.

This article defines a systematic ontology for debugging and validation — integrating hardware, software, and conceptual reliability checks into a single reproducible framework.

---

## 2. Context & motivation

Debugging closed-loop systems is uniquely difficult because **failures are often silent**:  
no crash, no visible error, yet the loop breaks causally.  

Common symptoms encountered in iterative setup testing:
- TTL outputs not appearing on DAQ lines.
- Camera triggers delayed or dropped despite correct routing.
- Arduino responses inconsistent or frozen.
- Bonsai execution appears “running” but data not updated.

These errors often stem from:
- Buffer misalignment in finite DAQ tasks.
- Thread contention between ObserveOn schedulers.
- Missed `.Start()` or `.Dispose()` calls before rearming.
- Hardware-level misrouting or grounding inconsistencies.

Motivation: To create a reproducible debugging schema that ensures **each failure is traceable**, **each fix is testable**, and **each system’s reliability is measurable**.

---

## 3. Observed behaviour & failure modes

| Symptom | Underlying cause | Verification method | Resolution |
|----------|------------------|--------------------|-------------|
| TTL not generated | Task auto-started before trigger | Use `ConsoleWrite` to print task state | Disable auto-start, use explicit `.Start()` |
| DAQ output stuck HIGH | Task not disposed between trials | Log DAQ state after `.Dispose()` | Recreate finite task on each OnNext() |
| Camera not synced | PFI routing miswired or floating ground | Oscilloscope probe on both ends | Verify shared reference ground |
| Bonsai node stalls | ObserveOn thread blocked by upstream | Attach Visual Studio debugger | Use separate NewThreadScheduler |
| Arduino misses pulse | Serial latency or event collision | Add timestamp print from Arduino | Use TTL instead of serial trigger |
| Data logs misaligned | Timestamp drift or buffer mismatch | Plot Δt histogram post-session | Standardize to DAQ clock timestamps |

---

## 4. Mechanistic analysis

Closed-loop experiments form a **temporal dependency graph**.  
Every node (Timer, Script, DAQ, Camera, Arduino) must obey specific life-cycle states:
1. **Create → Configure → Arm → Trigger → Dispose**

Failures often occur when this sequence breaks silently, e.g.:
- **Arm before Configure**: DAQ buffer undefined.
- **Trigger before Arm**: rising edge ignored.
- **Dispose after rearm**: task handle invalidated mid-run.

Real-world example from development:
> In one Bonsai flow, a `DigitalOutput` task was reused across trials without disposal.  
> The first trial succeeded, but subsequent ones showed no TTL output.  
> Visual Studio inspection revealed `DAQmx error -200088: Task not valid.`  
> Adding a `.Dispose()` and task recreation inside `OnNext()` fixed the issue permanently.

---

## 5. Implementation — practical debugging layers

### 5.1 Software-level logging
- Add **ConsoleWrite** nodes after critical operators (e.g. `StartTrigger`, `DigitalOutput`) to visualize runtime flow.
- Enable **Verbose logging** in Bonsai to surface hidden DAQmx errors.
- For C# source nodes: wrap every DAQ operation in `try/catch` and print task status codes.

### 5.2 Hardware-level verification
- Use **oscilloscope or logic analyzer** on PFI lines to confirm trigger arrival.
- For DAQ outputs, probe directly on connector pins to check voltage change.
- For camera, monitor *Exposure Active* or *Frame Valid* TTL signals.

### 5.3 Thread-level inspection
- Use `ObserveOn(NewThreadScheduler)` and verify thread independence using log timestamps.
- Avoid cross-thread race by ensuring each hardware stream runs on its own scheduler.
- Test by intentionally loading CPU and measuring task delay drift.

### 5.4 Real-time monitoring
- Implement an **LED mirror** for every critical TTL line (Arduino pin out → visible indicator).
- Visually confirm sequence: trigger → frame → valve → reset.

### 5.5 Validation cycles
1. **Unit validation:** Check each subsystem (DAQ / Camera / Arduino) independently.
2. **Pairwise validation:** Test two at a time (e.g. DAQ–Camera, DAQ–Arduino).
3. **Full loop validation:** Run end-to-end, log all timestamps, compare causal order.

---

## 6. Behavioural verification

### Case study: DAQ + Camera desynchronization
- Initial observation: DAQ trigger pulses visible on scope, camera frames delayed ~3 ms.
- Diagnosis: Exposure setting auto-adjusting in Vimba driver.
- Resolution: Fixed exposure time and forced external trigger mode.  
Result: Camera frame jitter reduced from 3 ms → 0.2 ms.

### Case study: Arduino dropped trigger
- TTL trigger verified on scope but no behavioral output.
- Diagnosis: Arduino interrupt handled during serial write (blocking ISR).
- Resolution: Separated serial and TTL handling on distinct pins, rewired external interrupt.
- Verification: Continuous TTL-to-LED blink confirmed stable for 30 min loop test.

---

## 7. Operational principles (design rules)

1. **Never assume DAQ tasks persist safely between trials.**  
   Always recreate finite tasks dynamically.

2. **Prefer hardware indicators for debugging over logs.**  
   LEDs and oscilloscopes do not lie.

3. **Treat every TTL as data.**  
   Record and timestamp it alongside behavior data.

4. **Reproduce bugs under controlled isolation.**  
   Disable all but one subsystem and observe change.

5. **Integrate validation into the pipeline.**  
   Make debugging an active node graph, not an afterthought.

6. **Log everything in the same temporal space.**  
   Timestamp camera, DAQ, and Arduino events relative to the same master trigger.

7. **Verify “idle state” between trials.**  
   Unreleased DAQ handles or high TTL lines often create phantom triggers.

---

## 8. Conceptual insight (temporal framing)

In a reactive system, debugging is **temporal archaeology** — reconstructing what failed, when, and in what order.  

Each error, buffer mismatch, or dropped trigger reveals the system’s **temporal topology**: the edges where asynchronous processes meet.  

From an ontological view, *failures are not noise* — they are **metadata about system reliability**, providing empirical evidence for the temporal structure of the experiment itself.

---

## 9. Classification (metadata)

- **Domain:** Systems Neuroscience / Experimental Control  
- **Software layer:** Bonsai (DAQmx, Custom C# nodes, Scheduler)  
- **Hardware layer:** NI DAQ, Vimba Camera, Arduino TTL  
- **Category:** Debugging, Validation, Reliability Analysis  

---

## 10. Summary & next steps

Robust closed-loop experiments are not achieved through perfect design, but through **iterative debugging and temporal introspection**.

**Next steps:**
- Build a standard “validation template” Bonsai workflow including LED mirrors, timestamp loggers, and state checks.
- Automate task recreation and disposal routines.
- Create a shared NeuroHardware debugging checklist for reproducibility across setups.

---

**NeuroHardware entry:**  
This document formalizes debugging as part of experimental ontology — positioning errors, verification steps, and recovery mechanisms as intrinsic components of closed-loop system design.
