---
layout: page
title: 4. QoS and Node Prioritization
permalink: /qos-ranking/
---

# 4. QoS and Node Prioritization

**IP-CRITICAL SECTION**

This section documents the novel **Proof-of-Quality (PoQ)** algorithm used for node selection. This algorithm is the primary differentiator from stake-based or reputation-based ranking systems used in blockchain networks.

---

## 4.1 Design Philosophy

### 4.1.1 Quality Over Quantity

Traditional distributed compute networks optimize for:
- **Lowest price** (race to the bottom)
- **Highest stake** (plutocratic bias)
- **Recent reputation** (vulnerable to manipulation)

Gap Junction instead optimizes for **predictable quality**, recognizing that enterprise workloads prioritize:
- **Reliability:** Tasks complete successfully.
- **Consistency:** Performance is predictable.
- **Capacity:** Resources are available when needed.

### 4.1.2 Fiat-Aligned Incentives

Because payments are in fiat (USD), the incentive structure differs from token-based systems:
- Nodes cannot "stake" their way to priority.
- Nodes cannot earn tokens by running low-quality infrastructure.
- Payment is directly proportional to **successful compute delivery**.

---

## 4.2 The Scoring Function

**NOVELTY CLAIM:** The following scoring function is believed to be novel as of the publication date.

### 4.2.1 Mathematical Definition

The utility score $U(n)$ for a node $n$ is defined as:

$$
U(n) = \sum_{i \in \{u, l, c, p, r\}} w_i \cdot S_i(n)
$$

Where:
- $S_u$: Uptime score
- $S_l$: Latency score
- $S_c$: Consistency score
- $S_p$: Capacity score
- $S_r$: Reliability score
- $w_i$: Weight for component $i$

### 4.2.2 Weight Distribution

| Component | Weight | Justification |
|-----------|--------|---------------|
| Uptime ($w_u$) | 0.30 | Historical availability is the strongest predictor of future availability |
| Latency ($w_l$) | 0.20 | Lower latency reduces overall task completion time |
| Consistency ($w_c$) | 0.20 | Consistent latency is more valuable than occasional fast responses |
| Capacity ($w_p$) | 0.15 | Available resources determine whether a task can run |
| Reliability ($w_r$) | 0.15 | Task completion rate predicts future success |

### 4.2.3 Component Score Functions

#### Uptime Score ($S_u$)

**Novelty:** Exponential penalty for uptime below threshold.

$$
S_u(n) = \begin{cases}
\frac{P_{up}}{100} & \text{if } P_{up} \geq 90\% \\
\left(\frac{P_{up}}{90}\right)^3 & \text{if } P_{up} < 90\%
\end{cases}
$$

Where $P_{up}$ is the 30-day rolling uptime percentage.

**Rationale:** The cubic penalty ensures that nodes with <90% uptime are severely disadvantaged. For example:
- 95% uptime → $S_u = 0.95$ (linear)
- 85% uptime → $S_u = (85/90)^3 = 0.84$ (slight penalty)
- 70% uptime → $S_u = (70/90)^3 = 0.47$ (severe penalty)

#### Latency Score ($S_l$)

$$
S_l(n) = \max\left(0, 1 - \frac{L}{L_{max}}\right)
$$

Where:
- $L$: Measured round-trip latency to Control Plane (ms)
- $L_{max}$: Maximum acceptable latency (500ms)

#### Consistency Score ($S_c$)

$$
S_c(n) = \max\left(0, 1 - \frac{\sigma_L}{\sigma_{max}}\right)
$$

Where:
- $\sigma_L$: Standard deviation of latency over last 100 samples
- $\sigma_{max}$: Maximum acceptable standard deviation (100ms)

**Novelty:** Rewarding consistency, not just raw speed. A node with 100ms ± 5ms is preferred over a node with 50ms ± 100ms.

#### Capacity Score ($S_p$)

$$
S_p(n) = \frac{M_{avail}}{M_{total}} \cdot \left(1 - \frac{CPU}{100}\right)
$$

Where:
- $M_{avail}$: Available memory (bytes)
- $M_{total}$: Total memory (bytes)
- $CPU$: Current CPU utilization (%)

#### Reliability Score ($S_r$)

$$
S_r(n) = \frac{T_{success}}{T_{total}}
$$

Where:
- $T_{success}$: Number of tasks completed successfully
- $T_{total}$: Total tasks assigned

---

## 4.3 Node Selection Algorithm

### 4.3.1 Selection Process

```
Algorithm: SelectNodes(count)
Input: count (number of nodes to select)
Output: List of (node_id, score) tuples

1. FILTER all nodes where last_heartbeat < NOW - 60 seconds
2. FOR each active node n:
   a. CALCULATE U(n) using the scoring function
   b. STORE (n.id, U(n)) in score_list
3. SORT score_list by score DESCENDING
4. RETURN first `count` entries from score_list
```

### 4.3.2 Caching Strategy

To avoid recomputing scores on every dispatch:
- Scores are cached for 10 seconds.
- Cache is invalidated on heartbeat for the affected node.
- Full cache refresh occurs every 60 seconds.

### 4.3.3 Blacklisting

Nodes that fail tasks are temporarily blacklisted:
- Duration: 5 minutes (configurable)
- Blacklisted nodes are excluded from selection.
- Blacklist is cleared after duration expires.

---

## 4.4 System Embodiment

The algorithm is implemented in:
- **File:** `/controller/internal/scheduler/qos.go`
- **Function:** `CalculateScore(node *Node) QoSScore`
- **Constants:** Defined at package level for transparency

```go
const (
    WeightUptime      = 0.30
    WeightLatency     = 0.20
    WeightConsistency = 0.20
    WeightCapacity    = 0.15
    WeightReliability = 0.15
    
    MaxLatencyMs     = 500.0
    MaxLatencyStdDev = 100.0
    MinUptimePercent = 0.90
)
```

---

## 4.5 Prior Art Differentiation

| System | Selection Mechanism | Gap Junction Novelty |
|--------|--------------------|-----------------------|
| Golem | Reputation + stake | No stake; exponential uptime penalty |
| Akash | Auction-based | No bidding; deterministic scoring |
| BOINC | First-come-first-served | QoS-ranked priority |
| AWS Spot | Price-based | Quality-based, fixed pricing |

The combination of:
1. **Exponential uptime penalty** (cubic function below threshold)
2. **Consistency reward** (standard deviation minimization)
3. **Fiat-bound identity** (Stripe verification)

...constitutes a novel system embodiment not present in prior art.
