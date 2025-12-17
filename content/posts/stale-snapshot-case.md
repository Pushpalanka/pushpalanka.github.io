---
title: "The Case of the Stale Snapshot: Debugging OPA Race Conditions"
date: 2025-12-17
tags: ["Engineering", "Debugging", "Multi-threading", "Go", "Race condition"]
summary: "How I tracked down a 'Phantom Error' in a high performance Authorization layer that defied logic, race detectors, and logs."
cover:
    image: "/images/multi-threading-case/multi-threading-case-1.webp"
    alt: "The Case of the Stale Snapshot"
---

It started as a simple optimization task: make Skipper start faster. It ended as a detective story involving a race condition that was "impossible."

Below is the full visual case study I presented to Concurrency and Operating Systems students. Full detailed blog can be found at [Beyond Race Detectors](https://pushpalanka.medium.com/beyond-race-detectors-first-hand-experience-debugging-a-multi-threaded-stale-data-issue-debd3a65da14).

### 📥 [Download as PDF](/stale-snapshot-case.pdf) (For offline reading)

---
{{< rawhtml >}}
<div class="slide-container">

    <img src="/images/multi-threading-case/multi-threading-case-1.webp" 
         alt="Presentation Title: The Case of the Stale Snapshot. Debugging a multithreaded mystery where the system lies to itself. By Pushpalanka Jayawardhana." 
         loading="eager">

    <img src="/images/multi-threading-case/multi-threading-case-2.webp" 
         alt="The Mission: Optimizing Zalando's Skipper Ingress Controller for faster startup using parallel OPA instance loading." 
         loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-3.webp" 
         alt="The Phantom Error: A 'Locked Room Mystery' in Kubernetes where a pod reports less than 5% error rate with zero logs." 
        loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-4.webp" 
         alt="Ruling out suspects: Why this was not a buggy OPA policy or corrupted bundle download." 
         loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-5.webp"
         alt="Reproduction Strategy: Forcing the bug out of hiding by increasing the scale of OPA bundles in the local setup." 
         loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-6.webp" 
         alt="The Impossible Transition: Logs revealing the Bundle Plugin flipping from OK back to NOT_READY without a valid code path." 
         loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-7.webp" 
         alt="The Smoking Gun: Logs proving the listener received a 'Stale Snapshot' message from the past, while the source of truth was already Healthy." 
         loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-8.webp" 
         alt="Digital Timeline Analysis: A visual diagram of the Race Condition. Routine A is preempted by the OS scheduler, waking up later to deliver stale data." 
         loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-9.webp" 
         alt="Code snippet that allowed stale snapshot"
         loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-10.webp" 
         alt="The Fix: Changing the code to bypass the notification channel and query the OPA Manager source of truth directly." 
         loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-11.webp" 
         alt="Lessons Learned: Why structured logs beat debuggers for concurrency, and why race detectors miss event ordering bugs." 
         loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-12.webp" 
         alt="Lessons Learned: Questioning assumptions and using AI as a productivity tool, not a debugger." 
         loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-13.webp" 
         alt="Beyond data races, to logic races. The hidden dangers of information propagation." 
         loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-14.webp" 
         alt="Case File Sources: Links to the GitHub Issue#8009, Skipper PR#3562, and original engineering blog posts." 
         loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-15.webp" 
         alt="Conclusion, with multi-threading time is not linear" 
         loading="lazy">

    <img src="/images/multi-threading-case/multi-threading-case-16.webp" 
         alt="thank you slide." 
         loading="lazy">



</div>

<style>
  .slide-container { max-width: 100%; margin: 20px auto; }
  .slide-container img { width: 100%; height: auto; display: block; margin-bottom: 0; border: none; }
</style>
{{< /rawhtml >}}
---

### Key Takeaways
1. **Logs > Debuggers:** You cannot step-through a race condition in an IDE.
2. **Time is not liner in a multi-threading system:** Parallel executions are not really parallel with pre-empting.
3. **The Quick Fix:** Bypass the notification channel and query the Source of Truth directly.
