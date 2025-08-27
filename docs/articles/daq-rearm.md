# The DAQ That Only Listens Once: Rearming Finite Tasks in Bonsai

## Introduction

You run your Bonsai graph. The DAQ fires. The waveform goes out. Everything looks perfect.  

Then you try it again — nothing.  

You double-check your graph. Nothing.  

You start wondering if your DAQ just died, or worse, if you’ve finally reached the mystical point where Bonsai decides to become self-aware and refuse your commands.  

What’s really happening? You’ve stumbled into the world of **finite sampling tasks in NI DAQmx** — where tasks work once and then silently give up unless you explicitly **rearm** them.  

This post is about how I discovered this quirk, what dead ends I chased, and how I finally convinced the DAQ to wake up every trial.  

---

## Background: Finite vs Continuous Sampling

With NI DAQ hardware, you can output data in two modes:

- **Continuous mode** → The buffer loops forever, DAQ just keeps spitting out data. Easy.  
- **Finite mode** → You give the DAQ a buffer of N samples. It plays them once, then stops.  

Bonsai’s DAQ nodes support both. But here’s the catch: **finite tasks do not automatically reset**. After one run, the task is “done” — like a one-shot firework.  

Unless you explicitly recreate or rearm the task, it will just sit there like a burnt-out fuse.  

---

## The Symptom

- First trial: DAQ outputs exactly what I want.  
- Second trial: **no output, no error, just silence.**  
- Third trial: silence again.  

The DAQ had become the world’s worst bandmate — plays the first note beautifully, then just stands there with crossed arms while the rest of the song goes on.  

---

## Debugging Attempts (and Dead Ends)

### 1. Buffer Size Experiments
I tried sending `[0, 1, 0, 1, ...]` sequences of different lengths. Sometimes I’d get a pulse, sometimes not. Eventually I learned:
- In **finite mode**, the buffer size must match your sample count exactly.  
- Extra samples? Task refuses to play them.  
- Too few samples? Same result.  

Useful to know, but didn’t fix the one-shot problem.  

---

### 2. Autostart On/Off
- With `autostart = true` → DAQ fired immediately when data arrived, but only once.  
- With `autostart = false` → Task armed but never started, unless triggered manually. Bonsai didn’t give me fine-grained control here.  

So either the DAQ jumped the gun, or it never left the locker room.  

---

### 3. Adding Delays (`Thread.Sleep`)
I thought maybe the DAQ just wasn’t ready in time. I tried inserting arbitrary delays between trigger and task setup.  
- Sometimes one node fired, sometimes none.  
- Never worked reliably across trials.  

Lesson: **delays don’t fix a DAQ that isn’t rearmed.**  

---

### 4. Creative Trigger Routing
I tried routing one output’s trigger into another PFI line, hoping one task could wake the other up.  
- Looked elegant on paper.  
- Failed in reality — because if a task isn’t armed, no trigger in the world will save it.  

---

## The Breakthrough: Rearm or Die

After digging through DAQmx docs (and poking around Bonsai with Visual Studio), I realised the problem:

- **Finite tasks don’t reset.** Once they’ve played their buffer, they’re finished.  
- Bonsai didn’t expose any way to “rearm” them between trials.  

The fix was obvious in hindsight: **I had to rebuild the task each time.**  

---

## The Fix: Custom Digital Output Node

I wrote a custom C# DigitalOutput node that:

- Recreates the DAQ task each time it receives new data  
- Flushes the old buffer  
- Disables `autostart` by default  
- Exposes a `.StartTrigger` property for external triggers  
- Explicitly calls `.Start()` inside `OnNext()`  
- Most importantly: includes a `Rearm = true` flag so the task is armed again every trial  

Result? The DAQ finally responded **every time**. First trial, second trial, hundredth trial.  

---

## Why This Works

The core issue was **task lifecycle**:

- In finite mode, a DAQ task ends after one buffer.  
- Unless it’s rearmed (or recreated), it’s dead.  
- Bonsai nodes didn’t handle this automatically.  

By explicitly recreating and rearming the task on each new buffer, I forced DAQmx to be ready every trial.  

---

## Lessons Learned

- **Finite ≠ Continuous:** In finite mode, sample count must match buffer size exactly.  
- **Silent Failures Hurt:** The DAQ won’t throw errors when it’s simply not rearmed — you’ll just see no output.  
- **Rearming is Essential:** Without it, tasks are one-shot only.  
- **Autostart Can Mislead:** It makes tasks fire once, but hides the deeper lifecycle issue.  
- **When in Doubt, Recreate the Task:** It’s safer to over-arm than under-arm.  

---

## Reflections

This issue cost me hours of head-scratching — and from what I’ve seen, a lot of Bonsai + DAQ users run into the exact same wall.  

If your DAQ works once and then sulks in silence, the reason is simple: you’re not rearming it.  

Give it a clean task, arm it properly, and it’ll play ball every time.  

---

 Next time, I’ll dive into how this rearming issue interacts with **synchronising analog and digital outputs**, and why threading (`ObserveOn`) became the second half of the solution.

