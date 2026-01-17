# Gap Junction: Fiat-Bridged Residential Compute Grid

## Technical Whitepaper v1.0

**Publication Date:** January 17, 2026  
**Status:** Prior Art Disclosure  
**License:** Apache 2.0 / CC BY 4.0

---

## Abstract

This document specifies **Gap Junction**, a distributed compute orchestration platform that routes WebAssembly (Wasm) workloads to residential compute nodes with settlement via traditional fiat payment rails. The system explicitly rejects cryptocurrency and blockchain mechanisms in favor of a **Credit/Threshold Aggregation Model** that enables sub-cent compute pricing while complying with payment processor constraints.

The primary innovations disclosed herein are:

1. **Proof-of-Quality (PoQ) Node Ranking** — A deterministic scoring algorithm prioritizing reliability over throughput
2. **Fiat-Bridged Aggregation Model** — Pre-paid credits with threshold-based PayPal payouts
3. **Stripe-Bound Identity Bootstrap** — mTLS certificates tied to verified payment accounts

This publication establishes prior art under 35 U.S.C. § 102.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Architecture](#2-system-architecture)
3. [Security Model](#3-security-model)
4. [QoS and Node Ranking](#4-qos-and-node-ranking)
5. [Financial Architecture](#5-financial-architecture)
6. [Legal Framework](#6-legal-framework)
7. [Implementation Reference](#7-implementation-reference)

---

## 1. Introduction

### 1.1 Problem Statement

The distributed computing landscape is dominated by two paradigms:

| Paradigm | Examples | Limitation |
|----------|----------|------------|
| Centralized Cloud | AWS, GCP, Azure | Expensive, vendor lock-in |
| Blockchain Networks | Golem, Akash, Render | Token volatility, gas fees, wallet complexity |

Neither paradigm effectively harnesses the **estimated 1.5 billion underutilized residential PCs worldwide**.

### 1.2 The Gap Junction Solution

Gap Junction bridges this gap by providing:

- **Fiat-Native Settlement:** All payments in USD via Stripe (renter) and PayPal (host)
- **Quality-First Selection:** Nodes ranked by reliability, not stake or price
- **Enterprise-Grade Security:** mTLS authentication, Wasm sandboxing

### 1.3 Key Differentiators

| Feature | Blockchain Alternatives | Gap Junction |
|---------|------------------------|--------------|
| Payment Rails | Native tokens | USD (Stripe/PayPal) |
| Price Stability | Market volatility | Fixed pricing |
| Node Selection | Stake/reputation | QoS-based |
| Per-Transaction Fees | Gas fees | None (aggregated) |
| Payout Complexity | Wallets, exchanges | PayPal email |

---

## 2. System Architecture

### 2.1 Component Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONTROL PLANE (Go)                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────────┐ │
│  │ gRPC API │ │ QoS      │ │ Billing  │ │ PostgreSQL +       │ │
│  │          │ │ Scheduler│ │ Ledger   │ │ TimescaleDB        │ │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └─────────┬──────────┘ │
│       └────────────┴────────────┴─────────────────┘            │
│                          │ mTLS                                │
└──────────────────────────┼─────────────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
    ┌────▼────┐       ┌────▼────┐       ┌────▼────┐
    │  AGENT  │       │  AGENT  │       │  AGENT  │
    │ (Rust)  │       │ (Rust)  │       │ (Rust)  │
    │ Wasmtime│       │ Wasmtime│       │ Wasmtime│
    └─────────┘       └─────────┘       └─────────┘
```

### 2.2 Technology Stack

| Component | Language | Key Dependencies |
|-----------|----------|------------------|
| Host Agent | Rust | wasmtime, tokio, rustls |
| Control Plane | Go | grpc-go, pgx |
| Database | PostgreSQL 16 | TimescaleDB extension |
| Renter Payments | Stripe | Card charges (prepay only) |
| Host Payouts | PayPal | Payouts API (email routing) |

### 2.3 Data Flow

1. **Renter** pre-pays credits via Stripe ($20 minimum)
2. **Renter** submits Wasm workload to Control Plane
3. **Control Plane** selects optimal node via QoS algorithm
4. **Agent** executes workload in Wasmtime sandbox
5. **Agent** reports fuel consumption to Control Plane
6. **Control Plane** deducts from renter ledger, credits host ledger
7. **Host** requests payout when balance ≥ $50 via PayPal

---

## 3. Security Model

### 3.1 Zero-Trust Architecture

All communication requires mutual TLS (mTLS) authentication:

- Agents present client certificates signed by the Platform CA
- Control Plane presents server certificate
- Certificate CN encodes node UUID for audit trail

### 3.2 Wasm Sandboxing

Guest workloads execute in Wasmtime with capability-based restrictions:

| Capability | Status | Rationale |
|------------|--------|-----------|
| Filesystem | DENIED | No host FS access |
| Network | DENIED | No raw sockets |
| Environment | DENIED | No env vars |
| Clock | READ-ONLY | Monotonic only |
| Random | ALLOWED | WASI random_get |

### 3.3 Resource Caps

| Resource | Limit | Enforcement |
|----------|-------|-------------|
| Memory | 256 MB | Wasmtime ResourceLimiter |
| CPU Time | 30 seconds | Epoch interruption |
| Instructions | 10B fuel | Fuel consumption |

---

## 4. QoS and Node Ranking

### 4.1 Design Philosophy

**NOVELTY CLAIM:** The Proof-of-Quality scoring function prioritizes node reliability over raw performance.

Traditional networks optimize for:
- Lowest price (race to the bottom)
- Highest stake (plutocratic bias)

Gap Junction optimizes for:
- Uptime consistency
- Latency stability
- Task completion rate

### 4.2 The Scoring Function

The utility score U(n) for node n is:

$$U(n) = w_u \cdot S_u + w_l \cdot S_l + w_c \cdot S_c + w_p \cdot S_p + w_r \cdot S_r$$

Where:

| Component | Weight | Description |
|-----------|--------|-------------|
| S_u (Uptime) | 0.30 | 30-day rolling uptime |
| S_l (Latency) | 0.20 | RTT to Control Plane |
| S_c (Consistency) | 0.20 | Latency standard deviation |
| S_p (Capacity) | 0.15 | Available memory × CPU headroom |
| S_r (Reliability) | 0.15 | Task success rate |

### 4.3 Exponential Uptime Penalty

**NOVELTY CLAIM:** Cubic penalty for uptime below 90%.

$$S_u(n) = \begin{cases} \frac{P_{up}}{100} & \text{if } P_{up} \geq 90\% \\ \left(\frac{P_{up}}{90}\right)^3 & \text{if } P_{up} < 90\% \end{cases}$$

Example impact:
- 95% uptime → S_u = 0.95 (linear)
- 85% uptime → S_u = 0.84 (slight penalty)
- 70% uptime → S_u = 0.47 (severe penalty)

### 4.4 Consistency Reward

**NOVELTY CLAIM:** Rewarding latency consistency over raw speed.

A node with 100ms ± 5ms is preferred over 50ms ± 100ms.

$$S_c(n) = \max\left(0, 1 - \frac{\sigma_L}{\sigma_{max}}\right)$$

---

## 5. Financial Architecture

### 5.1 The Aggregation Model

**NOVELTY CLAIM:** Credit/Threshold system that aggregates micropayments.

**REJECTED APPROACH:** Per-transaction payments
- Stripe fee: $0.30 + 2.9%
- $0.01 task → 3,003% fee overhead
- Economically non-viable

**ADOPTED APPROACH:** Aggregation Model

```
RENTER SIDE:                       HOST SIDE:
┌───────────────┐                  ┌───────────────┐
│ Stripe $20+   │                  │ Accumulate    │
│ (one charge)  │                  │ earnings      │
└───────┬───────┘                  └───────┬───────┘
        ▼                                  ▼
┌───────────────┐                  ┌───────────────┐
│ balance_cents │  ──per task──►  │ balance_cents │
│ (SQL ledger)  │                  │ (SQL ledger)  │
└───────┬───────┘                  └───────┬───────┘
        ▼                                  ▼
┌───────────────┐                  ┌───────────────┐
│ Deduct per    │                  │ PayPal payout │
│ task (SQL)    │                  │ when ≥ $50    │
└───────────────┘                  └───────────────┘
```

### 5.2 Thresholds

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Minimum Prepay | $20.00 | Amortize Stripe ~3% + $0.30 |
| Minimum Payout | $50.00 | Amortize PayPal ~2% |
| Platform Fee | 15% | Operational costs |
| Host Share | 85% | Maximize host incentive |

### 5.3 Fuel-to-Fiat Conversion

**Conversion Ratio:** 10,000,000,000 fuel = $0.01

This calibrates to approximately $0.001 per CPU-second.

### 5.4 PayPal Integration

**API:** PayPal Payouts (`POST /v1/payments/payouts`)

**Data Stored:** PayPal email address ONLY

**NOT Stored:** Bank account numbers, routing numbers, card details

**Rationale:** 
- No PCI-DSS compliance burden
- PayPal handles identity verification and FX
- Email is a routing address, not a credential

---

## 6. Legal Framework

### 6.1 Platform Neutrality (Section 230)

The Platform maintains Safe Harbor protection:

> "The Platform acts strictly as infrastructure. We do not inspect or endorse the algorithmic payloads executed by Renters."

### 6.2 Independent Contractor Status

Hosts are classified as independent contractors:
- No minimum hours
- No exclusive contract
- Hosts provide own equipment
- Hosts set own availability

### 6.3 Prohibited Use Policy

The following are explicitly banned:
- Cryptocurrency mining
- DDoS scripting
- Unauthorized penetration testing
- CSAM
- Malware distribution
- Export-controlled technology

### 6.4 Service Level Disclaimer

> "Gap Junction is a RESIDENTIAL compute grid, not a Tier-4 datacenter. We make NO GUARANTEE of uptime, availability, or latency."

---

## 7. Implementation Reference

### 7.1 Repository Structure

```
/gapjunction
├── /src
│   ├── /host-agent          # Rust Wasm host
│   └── /control-plane       # Go orchestrator
├── /docs
│   └── whitepaper.md        # This document
├── /migrations              # SQL schema
├── LICENSE                  # Apache 2.0
└── README.md
```

### 7.2 Key Source Files

| File | Purpose |
|------|---------|
| `host-agent/src/runtime/wasm_host.rs` | Wasmtime orchestration |
| `control-plane/internal/scheduler/qos.go` | QoS scoring algorithm |
| `control-plane/internal/billing/payouts.go` | Aggregation Model |

### 7.3 Environment Variables

| Variable | Description |
|----------|-------------|
| `PAYPAL_CLIENT_ID` | PayPal API credentials |
| `PAYPAL_CLIENT_SECRET` | PayPal API credentials |
| `STRIPE_SECRET_KEY` | Stripe API (renter prepays) |
| `GAPJUNCTION_DATABASE_URL` | PostgreSQL connection |

---

## Prior Art Disclosure

This document is published as a defensive disclosure under 35 U.S.C. § 102. The following claims are established as prior art:

1. **Claim 4.2:** QoS scoring function with weighted components
2. **Claim 4.3:** Exponential uptime penalty (cubic function)
3. **Claim 4.4:** Latency consistency reward
4. **Claim 5.1:** Credit/Threshold aggregation model
5. **Claim 5.4:** Email-only payout routing

**Publication Timestamp:** 2026-01-17T01:36:00-05:00

---

## License

**Document:** Creative Commons Attribution 4.0 (CC BY 4.0)  
**Code:** Apache License 2.0

---

*Gap Junction — Quality Compute, Fiat Settlement*
