# Cost Optimization in Cloud Infra

## Context

Spend had grown 40% YoY. No visibility into which workloads drove cost. Engineering teams had no cost awareness; requests were "add more capacity" without justification. Finance flagged cloud as top 3 OpEx line item.

## Problem

- No attribution: could not answer "how much does service X cost"
- Over-provisioned: most services ran at <20% CPU, some at <5%
- Unused resources: orphaned disks, idle dev clusters, forgotten staging envs

## Investigation

- Implemented cost allocation tags on all resources
- Ran utilization reports over 30 days
- Mapped spend to teams and services
- Identified top 10 cost drivers

Findings:
1. Dev/staging accounted for 35% of compute
2. Block storage from deleted VMs: ~$2k/month
3. Several services sized for Black Friday traffic year-round

## Tradeoffs

| Approach | Savings | Risk | Effort |
|----------|---------|------|--------|
| Right-size compute | High | Low | Medium |
| Reserved instances | Medium | Low | Low |
| Spot/preemptible | Very high | Medium | High |
| Kill dev overflow | High | Low | Low |

We prioritized quick wins first: cleanup, right-sizing, reserved instances for baseline. Spot for batch workloads only. Did not touch prod critical path until we had 6 months of utilization data.

## Solution

1. Cleanup: deleted orphaned resources, consolidated dev environments. Saved ~$1.5k/month.
2. Right-sizing: downsized 40+ instances based on P99 utilization. Saved ~$3k/month.
3. Reserved instances: 1-year commitments for baseline prod. Saved ~15% on those instances.
4. Governance: required cost tags on all new resources. Monthly review with team leads.

Total: ~25% reduction in 4 months. Projected annual savings: ~$80k.

## Lessons

- Cost visibility comes first. You cannot optimize what you cannot measure.
- Engineering buy-in required showing the tradeoff: savings fund new headcount and tools.
- Right-sizing is ongoing. We built a monthly report; teams self-correct when they see their numbers.
