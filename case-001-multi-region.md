# Multi-Region Reliability Design

## Context

A stateful service handling ~50k RPS across a single region. Business required 99.9% availability for EU customers. Single-region design meant any AZ or region outage translated to full customer impact. DR drill had never been run.

## Problem

- Single region = single point of failure for latency-sensitive workloads
- No automated failover
- Data residency requirement for EU meant we could not simply add a US region and call it DR

## Investigation

- Mapped all state: where it lived, how it replicated, RPO/RTO targets per dataset
- Ran failure-mode analysis: AZ loss, region loss, split-brain scenarios
- Evaluated active-passive vs active-active given consistency requirements

## Tradeoffs

| Option | RTO | Complexity | Cost |
|--------|-----|------------|------|
| Active-passive, async replicate | ~30 min | Medium | 1.5x |
| Active-passive, sync replicate | ~5 min | High | 2x |
| Active-active, eventually consistent | ~0 | Very high | 2.2x |

We chose active-passive with async replication. RTO of 30 minutes was acceptable for the use case. Sync replication added operational complexity we could not staff for. Active-active would have required application-level conflict resolution we did not have bandwidth to build.

## Solution

- Deployed standby region with async replication
- Built runbook for manual failover with decision tree
- Implemented health checks that could trigger alerting without auto-failover (manual gate)
- Ran quarterly DR drills, iterated on runbook

## Lessons

- DR is not a feature. It is a practiced discipline. First drill found 12 gaps in the runbook.
- Async replication meant we accepted up to 5 minutes of data loss on region failover. We documented this explicitly and got sign-off.
- Manual failover gate was intentional. We did not want automated failover on a false positive. The 30-minute RTO included human verification time.
