# Gap Junction API Specification

> ⚠️ **PRIOR ART NOTICE** ⚠️
> 
> These specifications are published to establish Prior Art under 35 U.S.C. § 102.
> Publication Date: January 17, 2026
> 
> Copyright (c) 2026 Angizmart. All rights reserved.
> You may reference these specifications for Prior Art purposes only.

---

## Files

| File | Format | Description |
|------|--------|-------------|
| [gapjunction.proto](./gapjunction.proto) | Protocol Buffers | gRPC service definitions |
| [billing.schema.json](./billing.schema.json) | JSON Schema | Billing API data structures |

---

## Prior Art Claims in This Directory

### 1. QoS Scoring Algorithm (gapjunction.proto)
- Weighted scoring formula with 5 components
- Exponential uptime penalty (cubic function below 90%)
- Consistency reward (latency standard deviation)

### 2. Aggregation Model (billing.schema.json)
- Renter pre-pay: $20.00 minimum
- Host payout threshold: $50.00 minimum  
- Fuel-to-cents ratio: 10B fuel = $0.01
- Revenue share: 85% host / 15% platform

### 3. Email-Only Payout Routing (billing.schema.json)
- PayPal email is the ONLY financial data stored
- No bank account numbers
- No routing numbers
- No card details

---

## Usage Restrictions

These specifications are provided for:
- ✅ Prior Art reference in patent disputes
- ✅ Educational understanding of the architecture
- ❌ NOT for implementation
- ❌ NOT for commercial use
- ❌ NOT for derivative works

See [LICENSE](../LICENSE) for full terms.
