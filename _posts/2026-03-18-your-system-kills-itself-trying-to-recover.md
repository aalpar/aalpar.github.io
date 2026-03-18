---
layout: post
title: "Your System Kills Itself Trying to Recover"
date: 2026-03-18
---

Distributed systems fail in two ways: the failure itself, and the system's automatic response to the failure. The second is usually deadlier.

Your Cassandra cluster comes back from a 10-minute outage. Hints replay. The replay saturates disk I/O, spiking read latencies, triggering read repairs, which consume more disk I/O — and the cluster is down again, this time from its own recovery mechanism. Your database server exceeds physical memory by 32 MB — 0.05% on a 64 GB machine — and thrashes to a halt. Not because the workload is unreasonable, but because the kernel's compensation for memory pressure amplifies the pressure.

These are routine failures. They're the predictable result of recovery mechanisms that no one tested against the stability condition they must satisfy. TCP figured this out 40 years ago. Most systems since have ignored it.

## TCP: recovery that contracts

TCP retransmission is the reference design for stable recovery. When a segment is lost, TCP retransmits — but each consecutive timeout doubles the wait. The retry rate halves with every failure.

The key property: every retransmission *reduces congestion on the link*. The recovery action undoes the condition that triggered it. Each round of backoff moves the system closer to equilibrium, not further. No load level makes TCP's recovery mechanism turn on itself. It is unconditionally stable — not because retransmissions are free, but because each one contracts the load.

And TCP knows where its authority ends. After about 15 minutes of failed retries, it refuses to guess what the failure means. It hands the application `ETIMEDOUT` and lets the layer with more context decide. TCP automates what it can prove. It surfaces what it cannot prove.

## Swap: the smooth collapse

TCP recovers on the same resource it contends for — link bandwidth — so recovery reduces contention. Swap crosses resource boundaries. Physical memory is full, so the kernel pages to disk. The system slows down but keeps running. Reasonable — until you compute the margin.

On a 64 GB database server with HDD, the system thrashes when the working set exceeds physical memory by 32 MB. That's 0.05%. NVMe buys you 16 GB of margin — 500 times more, proportional to the IOPS improvement. But the margin is finite, and no one monitors it.

The intuition: swap doesn't trade memory for disk *space*. It trades memory bandwidth for disk bandwidth. A memory access costs ~100 ns. A page fault to HDD costs ~10 ms. That's a 100,000x slowdown per access — and the only unit that makes both sides commensurable is time — the one resource no system budgets.

Every page fault consumes disk I/O. Disk saturation blocks processes. Blocked processes hold memory — and allocate more for pending work in their queues. More pages need swapping. The recovery action feeds back into the failure condition. Unlike TCP, there's a load level where this loop amplifies instead of contracting. That's the threshold. No production system I know of monitors this threshold against current load.

## Cassandra: the cliff

Swap's feedback grows with load — a smooth slide into failure. Cassandra's is a step function: zero gain below a threshold, catastrophic above it.

Cassandra's hinted handoff stores writes destined for a down node and replays them when it returns. Each replayed hint costs about one write — cheap in isolation. But hints accumulate linearly during the outage. A 10-minute failure at 100 Mbps per node produces roughly 7 GB of hints. At the default 1024 kbps throttle, that's about two hours of replay.

That's the gentle version. Here's the cliff.

Disk utilization during replay follows a queueing curve. At 99% utilization, read latency goes vertical — proportional to 1/(1 − utilization). Reads start timing out. Timeouts trigger read repairs. Each read repair costs additional disk I/O. More timeouts. More repairs.

On a 500-IOPS disk, the margin before this cascade is 5 IOPS — 1% of disk capacity. Below that threshold, the feedback gain is zero: no timeouts, no cascade. Above it, the gain jumps to roughly 3,000. Not a gradual degradation. A phase transition.

The default Cassandra hint replay throttle (128 ops/sec) stays well below this boundary. But operators under pressure raise the throttle. At 300 ops/sec, the system is past the cliff. The recovery mechanism meant to restore consistency instead destroys the cluster.

The contrast with swap matters: swap's boundary is smooth — you slide into thrashing as load increases. Cassandra's is a cliff — you're fine until you're not. Both boundaries are computable from measurable quantities. No one monitors either in practice.

## The pattern

These aren't three separate bugs. They're the same stability condition: does the recovery action amplify the failure, or contract it?

TCP contracts — each retry reduces congestion. Unconditionally stable. Swap and Cassandra amplify past a threshold, and both thresholds are closed-form functions of measurable parameters: disk IOPS, access rates, queue depths, working set sizes. A formal test — the spectral radius of a gain matrix — computes this boundary exactly for any system. TCP passes unconditionally. Swap and Cassandra pass until they don't.

The design principle: if your system can prove that its recovery bandwidth fits within available headroom, automate. If it cannot, emit state and shut down. Recover offline, where production load is zero and the bandwidth constraint is trivially satisfied.

Shutdown is recoverable. Silent cascade is not.

The full framework — recovery invariant, gain matrix derivations, stability proofs, and two more case studies — is in the paper: [*Don't Let Your System Decide It's Dead*](https://doi.org/10.5281/zenodo.19101786). The companion code (MATLAB/Simulink models that formalize and verify every claim) is on [GitHub](https://github.com/aalpar/failuredecider).
