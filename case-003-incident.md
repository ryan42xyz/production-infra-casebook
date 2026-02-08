# Production Incident Deep Dive

## Context

API service in prod. P99 latency normally 80ms. At 14:32 UTC, alerts fired: P99 > 2s, error rate 15%. Service served payments and checkout. Impact: customers unable to complete purchases.

## Problem

- Sudden latency spike and error rate increase
- No recent deployments
- Traffic pattern looked normal

## Investigation

1. **Check recent changes** — no deploys in 4 hours. Rolled back last deploy anyway. No improvement.
2. **Resource metrics** — CPU, memory, disk all normal. No throttling.
3. **Dependencies** — checked downstream services. One DB read replica showed elevated latency. But that replica was for analytics, not payments.
4. **Trace sampling** — pulled traces for failed requests. Saw pattern: slow calls to a specific external API (fraud check provider).
5. **External API** — their status page showed nothing. Our timeouts were 30s. Their P99 had degraded to 8s. Our workers were blocking on their responses.

Root cause: Third-party fraud check API degraded. We had no circuit breaker. Every payment request waited on their response. Queue backlog grew. Cascading failure.

## Tradeoffs

| Mitigation | Time to implement | Risk | Chosen |
|------------|-------------------|------|--------|
| Increase timeout | 5 min | Masks problem | No |
| Fail open (skip fraud check) | 30 min | Fraud risk | Yes, temporary |
| Circuit breaker | 2 hours | Need fallback path | Yes, permanent |

We failed open temporarily to restore service. Accepted elevated fraud risk for 45 minutes while we implemented circuit breaker with fallback to cached risk scores.

## Solution

1. **Immediate**: Feature flag to skip fraud check. Restored latency in 20 minutes.
2. **Short-term**: Implemented circuit breaker with 2s timeout, 5 failed requests = open circuit for 60s. Fallback: use cached risk score when circuit open.
3. **Long-term**: Added dedicated timeout (5s) for fraud check. Async fraud check where possible. Contract with provider for SLA and escalation path.

## Lessons

- External dependencies are single points of failure. Treat them like your own infra.
- Long timeouts on external calls will kill you. Default to 2–5s for user-facing paths.
- Circuit breakers are not optional for payment-critical paths. We had the pattern in our runbook but had not applied it to this integration.
- Post-incident: we added dependency health to our status page. Customers should know when a third party is degrading our service.
