<div align="center">

# 09. Security Architecture and SRE Observability

[![Security](https://img.shields.io/badge/Security-Zero%20Trust%20%2F%20TLS%201.3-blue?style=for-the-badge)]()
[![Observability](https://img.shields.io/badge/Telemetry-OpenTelemetry%20%2B%20Prometheus-orange?style=for-the-badge)]()
[![Compliance](https://img.shields.io/badge/Audit-Structured%20Audit%20Trail-success?style=for-the-badge)]()

[Back to Overview](../README.md) |
[Prev: 08 Scalability & Reliability](../08_Scalability_Reliability/README.md) |
[Next: 10 Architecture Decision Records ->](../10_ADR/README.md)

---

</div>

## Architecture Overview

```mermaid
graph TD
    subgraph SecurityShield["Defense-in-Depth Security"]
        S1["Edge WAF & DDoS Shield"]
        S2["API Gateway JWT Auth & RBAC"]
        S3["Token Bucket Rate Limiting"]
        S4["HashiCorp Vault Secret Management"]
    end

    subgraph ObservabilityPillars["The 3 Pillars of Observability"]
        O1["Metrics: Prometheus & Grafana"]
        O2["Logs: Structured JSON to ELK / Loki"]
        O3["Traces: OpenTelemetry & Jaeger"]
    end
```

---

## 1. Multi-Layer Security Architecture

| Security Layer | Threat Addressed | Enforcement Mechanism |
|---|---|---|
| **Edge / WAF** | DDoS, Bot scraping, SQLi, XSS | Cloudflare WAF managed rules, Challenge-response CAPTCHA |
| **API Gateway** | Unauthorized access, token forgery | RS256 signed JWT validation, Role-based Access Control (RBAC) |
| **Rate Limiter** | Brute force, flash-sale endpoint flooding | In-memory token bucket (Redis sliding window: 500 req/min) |
| **Data in Transit** | Man-in-the-Middle eavesdropping | Strict TLS 1.3 encryption with HSTS enabled |
| **Data at Rest** | Database snapshot exposure | AES-256 transparent data encryption (TDE) in PostgreSQL |
| **Secrets Mgmt** | Hardcoded credential leakage | HashiCorp Vault dynamic database & gateway credentials |
| **Audit Trails** | Non-repudiation & fraud analysis | Immutable append-only structured audit logs for all financial mutations |

---

## 2. Distributed Tracing Workflow

```mermaid
sequenceDiagram
    autonumber
    actor Customer as Shopper
    participant GW as API Gateway [traceparent: 4bf92f3577b34da6a3ce929d0e0e4736]
    participant INV as Inventory Service
    participant PAY as Payment Service
    participant ORD as Order Service
    participant LOG as OpenTelemetry Collector

    Customer->>GW: POST /api/checkout
    Note over GW: Injects Trace ID & Span ID
    GW->>INV: Step 1: Reserve Stock (Span ID: 00f067aa0ba902b7)
    INV->>LOG: Emit Span Telemetry (duration: 8ms)
    
    GW->>PAY: Step 2: Authorize Payment (Span ID: 5fb397be34d23b0f)
    PAY->>LOG: Emit Span Telemetry (duration: 320ms)
    
    GW->>ORD: Step 3: Persist Order (Span ID: a2fb4a1d1a96d312)
    ORD->>LOG: Emit Span Telemetry (duration: 14ms)
    
    GW-->>Customer: 200 OK (Total E2E Trace Time: 345ms)
```

---

## 3. SRE Alerting Thresholds and Monitoring Matrix

<details open>
<summary><b>Critical SRE Metrics and Alert Policies (Click to collapse)</b></summary>

| Metric Name | Target SLA | Warning Threshold | Critical Alert Threshold | Automated Action |
|---|---|---|---|---|
| **P99 API Latency** | < 200 ms | > 500 ms for 2 min | > 1500 ms for 1 min | Trigger Kubernetes HPA scale-out |
| **Payment Failure Rate** | < 2.0% | > 5.0% for 3 min | > 10.0% for 1 min | Trip Resilience4j Circuit Breaker |
| **Inventory Divergence**| 0 units | Stock mismatch detected | Stock mismatch detected | Freeze flash sale & alert on-call SRE |
| **Kafka DLQ Growth** | 0 msg | > 20 msg in 5 min | > 100 msg in 2 min | Page On-Call via PagerDuty |

</details>

---

[Return to System Overview](../README.md) | [Proceed to 10_ADR ->](../10_ADR/README.md)