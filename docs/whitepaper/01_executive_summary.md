---
layout: page
title: 1. Executive Summary
permalink: /executive-summary/
---

# 1. Executive Summary

## 1.1 Problem Statement

The distributed computing landscape is dominated by two paradigms:

1. **Centralized Cloud Providers** (AWS, GCP, Azure): Reliable but expensive, with vendor lock-in and geographic constraints.
2. **Blockchain-Based Compute Networks** (Golem, Akash, Render): Decentralized but suffer from cryptocurrency volatility, gas fee unpredictability, and complex tokenomics.

Neither paradigm effectively harnesses the **estimated 10+ billion underutilized residential compute devices** worldwide—laptops, desktops, and gaming PCs that sit idle for most of the day.

### 1.1.1 The Residential Compute Opportunity

| Metric | Estimate |
|--------|----------|
| Global PCs (2025) | 1.5 billion |
| Average Idle Time | 80% |
| Untapped Compute Capacity | 1.2 billion device-years annually |

### 1.1.2 Why Existing Solutions Fail

| Solution | Limitation |
|----------|------------|
| Cloud Providers | Cost prohibitive for burst workloads; egress fees |
| Blockchain Networks | Cryptocurrency volatility; gas fee unpredictability |
| Traditional P2P | No quality guarantees; no payment infrastructure |

## 1.2 The Gap Junction Solution

Gap Junction bridges this gap by providing:

1. **Fiat-Native Settlement:** All payments are in USD via Stripe Connect. No tokens, no wallets, no volatility.
2. **Quality-First Selection:** Nodes are ranked by reliability, not just price or stake.
3. **Enterprise-Grade Security:** mTLS authentication with identity bound to verified Stripe accounts.

### 1.2.1 Key Differentiators

| Feature | Gap Junction | Blockchain Alternatives |
|---------|--------------|------------------------|
| Payment Rails | Stripe (USD) | Native tokens |
| Price Stability | Fixed rates | Market-driven volatility |
| Node Selection | QoS-based | Stake/reputation-based |
| Identity | Stripe-verified | Wallet address |
| Sandbox | Wasmtime (capability-based) | Varies |

## 1.3 Target Use Cases

Gap Junction is optimized for **bursty, parallelizable workloads** where:

1. Latency tolerance is >100ms (not real-time).
2. Tasks are stateless and idempotent.
3. Cost efficiency matters more than absolute performance.

### Example Workloads

- **CI/CD Test Runners:** Parallel test execution across residential nodes.
- **Media Transcoding:** FFmpeg-compiled-to-Wasm for video processing.
- **Scientific Simulations:** Monte Carlo simulations, parameter sweeps.
- **Machine Learning Inference:** Edge inference with Wasm-based ML runtimes.

## 1.4 Business Model

Gap Junction operates as a **compute marketplace** with the following economics:

| Actor | Payment Flow |
|-------|--------------|
| **Client** | Pays per compute unit (fuel) consumed |
| **Node Operator** | Receives 85% of client payment |
| **Platform** | Retains 15% as service fee |

All payments are processed via Stripe Connect, with:
- Instant payouts available for verified nodes.
- Automatic tax form generation (1099-K).
- Built-in dispute resolution.

## 1.5 Prior Art Claims

This whitepaper establishes prior art for the following innovations:

1. **Section 3.1:** Stripe-bound mTLS identity bootstrap.
2. **Section 4.2:** Proof-of-Quality scoring algorithm with exponential uptime penalty.
3. **Section 5.2:** Fiat micropayment aggregation with threshold-based batch settlement.

These claims are made as of January 17, 2026.
