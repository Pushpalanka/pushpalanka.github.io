---
title: "Beyond Race Detectors: First-hand Experience Debugging a Multi-threaded Stale Data Issue"
date: 2025-11-11T00:00:00Z
draft: false
author: "Pushpalanka"
tags: ["go", "concurrency", "debugging", "multi-threading", "opa", "skipper", "zalando"]
categories: ["Technical"]
description: "Sharing first hand experience with OPA's plugin manager, demonstrating a snapshot-based, inconsistent notification problem"
cover:
    image: "/images/beyond-race-detectors/detective.webp"
    alt: "Multi-threading debugging illustration"
canonicalURL: https://pushpalanka.com/posts/beyond-race-detectors    
---

During a new feature rollout of Skipper (An opensource ingress controller from Zalando), we hit a puzzling issue: <5% of requests to a specific route failed consistently in one pod, while the same configuration worked perfectly everywhere else. The culprit? A timing-dependent bug in how OPA's plugin manager handles state notifications - one that Go's race detector won't catch.

*(If a more visual representation is appealing to you, refer the slide deck shared at https://pushpalanka.com/posts/stale-snapshot-case. It was prepared as an industry example for concurrency and operating system students.)*

## 1. Overview

This post explores a snapshot consistency problem in notification systems. It only appeared when starting 50+ OPA instances in parallel, revealing how resource contention can expose narrow timing windows in seemingly safe code. In this post, I'll share the journey of detecting, reproducing, isolating, and fixing a multi-threading issue that was both difficult to reproduce and even harder to pinpoint.

The issue I'll reference is [OPA issue #8009](https://github.com/open-policy-agent/opa/issues/8009), which surfaced when parallel plugin startups were occurring in Zalando Skipper. This integration is explained in details in [this Zalando blogpost](https://opensource.zalando.com/skipper/).

While this post builds on that specific case, the focus is on the general learnings and practical lessons that apply to debugging complex multi-threaded behavior in real-world systems.

## 2. Scenario

We released an optimization to the OPA filter creation mechanism in Skipper, that serves as the Ingress controller in our platform. The goal was to achieve faster startup times during large-scale deployments.

To do this, we modified the system to start all required OPA instances in parallel, allowing Skipper to become operational more quickly. Any additional OPA instances needed later, would be initialized on-demand in the background, one at a time, as required by the traffic route definitions.

Here I share the problem-contributing code segments. If you spot the issue at a glance, congratulations!, you'll save yourself hours debugging similar multi-threading issues.

### Notifier (from OPA plugins)

When a plugin's status updates, OPA creates a snapshot of all plugin states and notifies listeners,

```go
// UpdatePluginStatus updates a named plugins status. Any registered
// listeners will be called with a copy of the new state of all
// plugins.
func (m *Manager) UpdatePluginStatus(pluginName string, status *Status) {
 var toNotify map[string]StatusListener
 var statuses map[string]*Status

 func() {
  m.mtx.Lock()
  defer m.mtx.Unlock()
  m.pluginStatus[pluginName] = status
  toNotify = make(map[string]StatusListener, len(m.pluginStatusListeners))
  maps.Copy(toNotify, m.pluginStatusListeners)
  statuses = m.copyPluginStatus()
 }()

    // 🔴 Lock released here - listeners called concurrently
 for _, l := range toNotify {
  l(statuses)
 }
}
```

### Listener (from Skipper)

In each OPA instance running in Skipper, it listens to plugin status changes. At the start up, all OPA instances and their plugins go through state changes when getting active.

```go
 manager.RegisterPluginStatusListener("instance-health-check", func(status map[string]*plugins.Status) {
  opa.healthy.Store(allPluginsReady(status, bundle.Name, discovery.Name))
  opa.Logger().Info("OPA instance health updated: healthy=%t status=%+v", opa.healthy.Load(), status)
 })
```

Now I will share the story of its detection, the struggle and solving.

## 3. Issue Detection

During staging tests, one of our stakeholder teams reported a consistent error rate of less than 5% on specific requests. The errors were limited to a single route protected by a particular OPA policy and, intriguingly, only in one specific pod.

The same OPA policy worked perfectly fine in all other pods, and no other OPA policies showed any issues. Even more puzzling: there were no explicit error logs despite thorough error logging, except for a log indicating that the affected OPA instance became healthy for a fraction of a second, and then immediately unhealthy again and never recovers.

## 4. Reproducing

We went through the relevant code again and again, critically assessing what could have gone wrong, reviewed the end to end scenarios tests to see if there was anything we could cover more which didn't give any clue. Once these basic verifications are done, we could rule out few things.

- It can't be anything related to OPA policy issues - because the policy works in every other pod
- OPA bundle downloading issues - Logging helped ruling this out. If a download issue or a malformed bundle was present, it does log an error

Beyond this, only path we saw was reproducing the issue so we can have a closer look. I also tried different analyse paths with AI coding assistants(Claude Sonnet 4.5 and GPT 5), providing both Skipper and relevant OPA code, which didn't lead to anything solid, but lot of vague possibilities.

### Replicate everything as close as possible to the issue occurred environment

1. I used the same OPA bundles as our stakeholders and created a local setup. Made the configurations exact by splitting the data and policy bundles and including the discovery bundles in the configurations.
2. Configured status and decision log reporting to a local server where I can control them to simulate being slow, totally unavailable and recovering. Tried out variety of timing scenarios involving these and bundle server availabilities and slowness.

### Tools to analyse logs

Knowing whether the issue was reproduced or not was also a challenge. This is where AI could effectively help by providing the commands to isolate log patterns. In the issue occurred setup, there were close to 32 OPA instances getting created in parallel.

Meanwhile the stakeholder agreed to try it once more to gather anything that would help reproducing. This uncovered few more details. Even-though it didn't get reproduced in my local setup it was reproducing again in the stakeholder's setup, now failing for two OPA instances in one particular pod.

This confirmed two things,

- Failure is not specific to a particular OPA policy, but possibly can happen for any
- Something specific to this stakeholder's setup is making it quite easy for the issue to happen

With this newly revealed information, I kept on increasing the number of bundles in my local setup. I was going beyond what our stakeholder's had and as I go beyond 50 OPA instances, the mysterious issue start to happen consistently. I had the nice command I got from Co-pilot, confirmed it will isolate me the OPA instances that faced the failure and started analyzing the logs.

## 5. Debugging

This is where things got more interesting.

With increased logging, I could finally see more internal activity, but still no actual errors. Among the OPA plugins, I noticed that the bundle plugin was flipping from the OK state back to NOT_READY, which in turn triggered the brief healthy → unhealthy transition in OPA's overall health status.

Since I never knew which exact OPA instance would show this behavior ahead of time, logging became my friend, traditional step debugging in the IDE wasn't practical. So I relied heavily on structured logs, they became my eyes inside the concurrency maze.

The mystery deepened further:

> there was no code path that actually transitioned the bundle plugin from OK to NOT_READY between those health flips.

That raised a fundamental question:

**How could our health listener observe the bundle plugin switching back to NOT_READY when no execution ever performed that state change?**

### The Breakthrough

{{< figure src="/images/beyond-race-detectors/race.webp" caption="Two Go routines racing with each other, one with stale data" width="400" align="center" >}}

I added one critical log line:

```go
 for _, l := range toNotify {
  m.logger.Info("UpdatePluginStatus: plugin status listener %+v  status manager %+v", statuses, m.pluginStatus)
  l(statuses)
 }
```

Which printed,

```
"UpdatePluginStatus: plugin status listener map[bundle:{**NOT_READY** \"\"} decision_logs:{OK \"\"} discovery:{OK \"\"} envoy_ext_authz_grpc:{OK \"\"} status:{OK \"\"}]  
status manager map[bundle:{**OK** \"\"} decision_logs:{OK \"\"} discovery:{OK \"\"} envoy_ext_authz_grpc:{OK \"\"} status:{OK \"\"}]" bundle-name=bundles/discovery6.tar.gz
```

**The race:**

### The Timeline: Two Routines, One Stale Snapshot

| Time | Routine A | Routine B |
|------|-----------|-----------|
| **T1** | `UpdatePluginStatus("bundle", NOT_READY)` | |
| **T2** | Lock → Update → Create snapshot<br/>**S1 = {NOT_READY}** → Unlock | |
| **T3** | ⚠️ Preempted before calling listeners | |
| **T4** | | `UpdatePluginStatus("bundle", OK)` |
| **T5** | | Lock → Update → Create snapshot<br/>**S2 = {OK}** → Unlock |
| **T6** | | Call listeners with snapshot **S2** ✅ |
| **T7** | Resumes — Calls listeners with old snapshot **S1** ❌ | |

**Result:**

Routine A's delayed listener call overwrote Routine B's newer state, making the system *appear* to revert to `NOT_READY` — even though it was already healthy.

*Timeline with routines getting pre-empted*

## 6. Fixing

Once isolated we had a one-line fix that could be done in Skipper side. Instead of trusting the snapshot passed to the listener, directly query the manager's current state.

```go
manager.RegisterPluginStatusListener("instance-health-check", func(_ map[string]*plugins.Status) {
  // Get fresh status to workaround OPA issue https://github.com/open-policy-agent/opa/issues/8009
  status := opa.manager.PluginStatus() // <- fix, overrrides the pass in status map to the listener
  opa.healthy.Store(allPluginsReady(status, bundle.Name, discovery.Name))
  opa.Logger().Info("OPA instance health updated: healthy=%t status=%+v", opa.healthy.Load(), status)
 })
```

**Why this works:** We bypass the snapshot race by always reading the real-time state from the source of truth (the manager), rather than relying on potentially stale snapshots delivered via notifications.

## 7. Learnings

### 1. AI as a Productivity Tool, Not a Debugger

AI models aren't mature enough yet to analyze complex multi-threading issues, but they excel at accelerating the process by generating log parsing commands, documenting findings, and explaining unfamiliar codebases. The actual debugging still requires human intuition.

### 2. Strategic Logging Over Verbose Logging

Good logging saves time, but there's a balance. Log state transitions with context, not every operation. Key questions will be, "Will this log help confirm or rule out something critical? How often will it fire?" Debug logging everywhere reduces readability and buries the signal in noise.

### 3. Every Bug is Reproducible (With Enough Persistence)

This seemed impossible to reproduce initially. By systematically varying parameters, replicating the environment, and investing consistent effort over days, I could find conditions to reliably reproduce it. Reproducibility isn't binary, it's about finding the right conditions. When stakes are high, persistence pays off.

### 4. Race Detectors Catch Data Races

Go's race detector excels at finding unsynchronized memory access but didn't detect event ordering races like this, logical races in state machines, or snapshot consistency problems. Multi-threading issues are often about event ordering and state consistency, not just data races. Manual reasoning is essential.

### 5. Logs Beat Debuggers for Concurrent Bugs

When issues reproduce randomly and concurrency is suspected, logging is more effective than IDE debugging. You can't step through race conditions. To compensate for losing interactive stack traces, print stack traces directly in logs at critical state transitions for debugging purposes.

### 6. Question Your Assumptions Explicitly

Write down what you believe to be true and verify each assumption:

- "NOT_READY" means broken" → Actually: Real state was OK, snapshot was stale
- "All state changes notify" → Actually: Some state changes are protected by `sync.Once` so not all updates are notified

Half the battle is realizing what you thought to be true isn't.

## Conclusion

This bug broadened my horizons, that multi-threading issues aren't always about locks and data races, they are about information propagation and timing assumptions as well.

### Full technical details

- [Skipper Issue #3687](https://github.com/zalando/skipper/issues/3687)
- [Skipper PR #3692 Workaround](https://github.com/zalando/skipper/pull/3692)
- [Skipper PR #3562](https://github.com/zalando/skipper/pull/3562) (Preloading feature that exposed the bug)
- [OPA Integration in Skipper](https://opensource.zalando.com/skipper/)
- [OPA issue report #8009](https://github.com/open-policy-agent/opa/issues/8009)

Cheers! 🎯 ✨

---
