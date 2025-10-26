# Synchronising NI DAQ Analog and Digital Output in Bonsai Without LabVIEW

## 1. Concept definition

**Goal:** start analog and digital outputs on National Instruments (NI) DAQ hardware from Bonsai, using a single shared hardware trigger (PFI1), without LabVIEW — achieving consistent trial-to-trial behaviour for closed-loop experiments.

**Required signals in this workflow:**
- Digital output (TTL) — e.g., a square wave to trigger a camera or device.
- Analog output — e.g., white noise, ramp, or sine stimulus.
- Shared start trigger routed to **PFI1**.

Target behaviour: both outputs should begin on the same rising-edge trigger and be re-armable across repeated trials.

---

## 2. Context & motivation

Real-time synchronisation of multiple data streams and triggers is central to closed-loop systems neuroscience. NI DAQs provide flexible hardware routing and precise timing—but the software layer (DAQmx wrappers in Bonsai) abstracts many details and hides failure modes. Running DAQ output tasks in finite sampling mode and coordinating them via Bonsai’s reactive graph exposes mismatches between DAQmx expectations and Bonsai’s default execution model.

This note documents the problems encountered, the debugging process (including Visual Studio attachment), and the eventual solution combining custom C# nodes and Bonsai threading (`ObserveOn` + `NewThreadScheduler`). The aim is a reproducible, conceptually clear entry for the NeuroHardware corpus.

---

## 3. Observed behaviour & failure modes

### 3.1 Initial setup
- Waveforms generated in Bonsai `Script` nodes:
  - TTL waveform: e.g. `[0,1,0,1,...]` with **10,000 samples** at **1 kHz** (finite mode).
  - Analog waveform: white noise sampled at **1 kHz** for **15 seconds** (finite mode).
- Both `AnalogOutput` and `DigitalOutput` nodes assigned the same `.StartTrigger = /Dev1/PFI1`.
- Expected: both tasks arm and start on the same trigger.

### 3.2 Actual result
- Only one output executes on trigger; the other remains silent.
- The active output tends to be the one declared first in the graph.
- No immediate Bonsai error dialogs — failures were silent.

### 3.3 Debugging via Visual Studio
Attaching Visual Studio to Bonsai while running revealed:
- DAQ task objects were **not being recreated** per trial.
- Buffers were not flushed between trials.
- Bonsai’s `autostart` parameter produced uncontrolled one-shot behaviour.
- There was **no implicit rearm** — repeated triggers did nothing unless the task was reset/ recreated.

---

## 4. Mechanistic analysis

| Component | Observed constraint | Root cause |
|----------:|---------------------|-----------|
| **DigitalOutput (finite)** | Requires exact buffer length `{1, N}` where N == sample count | DAQmx fails silently if buffer/sample mismatch occurs |
| **AnalogOutput (finite)** | One-shot assumption; not rearmed automatically | Bonsai’s wrapper doesn’t recreate or rearm DAQmx tasks |
| **Trigger routing (PFI1)** | Only one task sees trigger when both share PFI1 | Bonsai schedules node execution serially so only one node arms in time |
| **Timing model** | Bonsai executes nodes synchronously within a stream | Serial arming leads to race conditions for hardware start triggers |

**Key insight:** Bonsai’s *reactive* appearance masks a synchronous execution backbone. Two nodes subscribing to the same trigger will not necessarily *arm* simultaneously; they must be prepared (armed) before the trigger edge reaches the DAQ hardware.

---

## 5. Implementation — steps taken to reconstruct synchrony

### 5.1 Custom Digital Output node (C#)
I implemented a custom Bonsai node with these behaviors:
- Accepts a 1-D OpenCV `Mat` as input (shape `{1, N}` for N samples).
- **Disables `autostart`** and explicitly controls `.Start()`.
- **Recreates the DAQmx task** for each incoming waveform (task recreation on `OnNext()`).
- Exposes `.StartTrigger` (set to `/Dev1/PFI1`).
- Supports `Rearm = true` so the node can be used across trials.
- Parameters for sample rate and buffer size configurable from node properties.

**Why:** finite sampling mode requires precise buffer/sample alignment and fresh task setup per trial; task recreation prevents stale handles and ensures DAQmx accepts the next trigger.

### 5.2 Analog Output adjustments
- Used the same re-creation and `autostart=false` pattern on the analog node.
- Waveform generation performed in a `Script` node (white noise or ramp), then passed to the modified AO node.
- Ensured sample rate and buffer count matched exactly to avoid silent failures.

### 5.3 Concurrent arming with `ObserveOn` / threading
Bonsai’s default execution is serial. To guarantee both AO and DO were armed before the shared trigger:

- Inserted `ObserveOn` operators before each output node.
- Externalized the scheduler and used `NewThreadScheduler` so each output runs on its own thread.
- Ensured each output thread performs the task creation and `.Start()` call independent of the main graph thread.

**Graph sketch:**

Timer → Script (white noise) → ObserveOn → AnalogOutput  
  ↘ Script (TTL) → ObserveOn → DigitalOutput  

Trigger Source → RisingEdge → DO line → PFI1  
Both AO and DO use **PFI1** as **StartTrigger**

# Synchronising AO and DO in Bonsai (Finite Sampling Mode)

## 5.4 Additional experiments attempted (and why they failed)
- **Cascading triggers (AO exports to PFI2 → DO):** failed because AO did not start early enough to generate the export (catch-22).
- **Adding `Thread.Sleep()` or `WaitUntilDone`:** ineffective — only masked the race, did not ensure reliable arming.
- **Relying on software triggers alone:** proved brittle compared to physical line routing.

---

## 6. Behavioural verification

- For hardware triggering, I used a Bonsai-scripted rising edge on a digital line physically wired to **PFI1**.  
- Qualitative verification across many trials showed both outputs beginning together on the trigger when using the custom nodes + `ObserveOn` threading configuration.  
- Observed behaviour: TTL trains and analog waveform started as expected, with consistent trial-to-trial execution.

> **Note:** Quantitative jitter and latency measurements (oscilloscope traces, timestamped DAQ logs) are planned for a follow-up post; current validation is functional and trial-consistent.

---

## 7. Operational principles (design rules)

1. **Finite-mode tasks require exact buffers:** In finite sampling, match `buffer size == sample count` exactly. Oversized or undersized buffers cause silent failures.  
2. **Recreate DAQ tasks per trial:** Do not reuse finite-mode tasks; recreate them in `OnNext()` to ensure rearm and fresh handles.  
3. **Disable `autostart`:** Use explicit `.Start()` after initialising the task to control the exact arming moment.  
4. **Use thread-level isolation for concurrent arming:** `ObserveOn(NewThreadScheduler)` gives each output its own thread context and prevents serial-arming race conditions.  
5. **Prefer hardware routing for determinism:** Physically wiring a digital line into PFI1 is more reliable than purely software-triggered signals.  
6. **Avoid `Thread.Sleep()` for synchronization:** Sleep/delay hacks do not guarantee proper arming; they only add nondeterministic timing windows.  
7. **Catch silent failures via active debugging:** Bonsai may not surface DAQmx errors; attach Visual Studio or add logging (`ConsoleWrite`) to detect task state issues.

These are not ephemeral debugging tips — they form the operational core of building robust closed-loop rigs with Bonsai + NI hardware.

---

## 8. Conceptual insight (temporal framing)

Synchronisation is a **three-clock alignment problem:**
- **Hardware time** — the DAQ’s internal engine (PFI routing, timing engine).  
- **Software time** — Bonsai’s reactive scheduler and node execution semantics.  
- **Cognitive time** — the experimenter’s mental model of event ordering and causality.  

A reliable system aligns all three: tasks must be prepared (software), routes must be correctly wired (hardware), and the experimenter must conceive triggers and trial structure coherently (cognitive). When these clocks are misaligned, behavior appears intermittent or silent. Treat each output node as a *temporal agent* that must be explicitly prepared and synchronized.

---

## 9. Classification (metadata)

- **Domain:** Systems Neuroscience / Experimental Control  
- **Software layer:** Bonsai (DAQmx nodes, Script, ObserveOn, Scheduler)  
- **Hardware layer:** NI DAQ (PFI1, PFI2, AO/DO lines)  
- **Category:** Synchronisation — Finite Sampling, Triggering, Concurrency  

---

## 10. Summary & next steps

Synchronising AO and DO in Bonsai without LabVIEW is feasible but requires:
- explicit DAQ task lifecycle management (recreate + `.Start()`),
- exact buffer/sample alignment in finite mode,
- and explicit thread separation for concurrent arming (`ObserveOn` + `NewThreadScheduler`).

**Next practical steps:**
- Share the **C# source** for the custom DO/AO nodes (planned follow-up).  
- Add **quantitative verification** (oscilloscope traces and jitter histograms).  
- Convert the config into a reproducible Bonsai graph file and a minimal reproducible example.

---

**NeuroHardware entry:**  
This document is intended to be a canonical node in the **NeuroHardware lexicon** — it captures a repeatable synchronization pattern (failure modes, mechanism, and design rules) that is applicable to other DAQ + visual-pipeline integrations.


