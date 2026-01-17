---
layout: page
title: 5. Fiat Settlement
permalink: /fiat-settlement/
---

# 5. Fiat Settlement

**IP-CRITICAL SECTION**

This section documents the **Aggregation Model** for fiat settlement, which explicitly rejects per-transaction payment processing in favor of credit-based aggregation.

---

## 5.1 The Protocol Model vs. Aggregation Model

**STRATEGIC DECISION:** Gap Junction adopts the Aggregation Model.

### 5.1.1 Rejected: Protocol Model (Per-Transaction)

The Protocol Model charges payment fees per compute task:

| Transaction Size | Stripe Fee | Fee % |
|------------------|------------|-------|
| $0.01 task | $0.30 + 2.9% | **3,000%** |
| $0.10 task | $0.30 + 2.9% | **303%** |
| $1.00 task | $0.30 + 2.9% | **33%** |

**Conclusion:** Per-transaction fees destroy margins for micropayments.

### 5.1.2 Adopted: Aggregation Model (Pre-Pay + Threshold Payouts)

The Aggregation Model separates payment events from compute events:

```
RENTER SIDE:                        HOST SIDE:
┌─────────────────┐                 ┌─────────────────┐
│  Pre-Pay $20+   │                 │  Accumulate     │
│  (Stripe once)  │                 │  Earnings       │
└────────┬────────┘                 └────────┬────────┘
         │                                   │
         ▼                                   ▼
┌─────────────────┐                 ┌─────────────────┐
│  Credit Ledger  │    ───────►    │  Balance >= $50 │
│  (balance_cents)│    per task    │  (balance_cents)│
└────────┬────────┘                 └────────┬────────┘
         │                                   │
         ▼                                   ▼
┌─────────────────┐                 ┌─────────────────┐
│  Deduct per     │                 │  On-Demand      │
│  task (SQL)     │                 │  PayPal Payout  │
└─────────────────┘                 └─────────────────┘
```

**Benefit:** One Stripe charge ($20) covers 20,000+ micro-tasks. One PayPal payout ($50) aggregates 50,000+ micro-earnings.

---

## 5.2 Renter Credit System

### 5.2.1 Pre-Pay Model

**NOVELTY CLAIM:** Minimum prepay threshold for credit-based compute billing.

| Constraint | Value | Rationale |
|------------|-------|-----------|
| Minimum Prepay | $20.00 USD | Amortize Stripe's ~3% + $0.30 fee |
| Credit Unit | 1 cent ($0.01) | Granular enough for micropayments |
| Fuel Ratio | 10B fuel = 1 cent | ~$0.001 per CPU-second |

### 5.2.2 Usage Deduction Algorithm

```
Algorithm: DeductUsage(renter_id, fuel_consumed)

1. amount_cents = floor(fuel_consumed / 10,000,000,000)
2. IF amount_cents == 0 AND fuel_consumed > 0:
       amount_cents = 1  // Minimum 1 cent for any compute
3. IF renter.balance_cents < amount_cents:
       REJECT "Insufficient balance"
4. renter.balance_cents -= amount_cents
5. RETURN success + charge_record

NOTE: No external API calls. Pure SQL transaction.
```

### 5.2.3 Balance Visibility

Renters can view their balance at any time:
- Current balance (cents)
- Total credits purchased (lifetime)
- Total credits spent (lifetime)
- Recent transactions (last 30 days)

---

## 5.3 Host Payout System

### 5.3.1 Accumulation Model

**NOVELTY CLAIM:** Threshold-based on-demand payouts via PayPal email routing.

| Constraint | Value | Rationale |
|------------|-------|-----------|
| Minimum Payout | $50.00 USD | Minimize PayPal's 2% fee impact |
| Payout Method | PayPal Email | No bank account storage (no PCI-DSS) |
| Payout Trigger | On-Demand | User clicks button (no automatic) |
| Currency | USD only | PayPal handles FX (we never convert) |

### 5.3.2 Earnings Credit Algorithm

```
Algorithm: CreditEarnings(host_id, fuel_consumed)

1. gross_cents = floor(fuel_consumed / 10,000,000,000)
2. IF gross_cents == 0 AND fuel_consumed > 0:
       gross_cents = 1
3. platform_fee = floor(gross_cents * 15 / 100)  // 15% take
4. net_cents = gross_cents - platform_fee        // 85% to host
5. host.balance_cents += net_cents
6. RETURN success + earnings_record
```

### 5.3.3 Payout Request Algorithm

```
Algorithm: RequestPayout(host_id)

1. IF host.balance_cents < 5000:  // $50.00 minimum
       REJECT "Minimum payout is $50.00"
2. IF host.paypal_email IS NULL:
       REJECT "No PayPal email on file"
3. payout_amount = host.balance_cents
4. host.balance_cents = 0  // Atomic reset
5. CALL PayPal Payouts API (batch of 1)
6. IF API fails:
       host.balance_cents += payout_amount  // Rollback
       REJECT "Payout failed, try again"
7. RETURN success + payout_record
```

---

## 5.4 PayPal Integration

### 5.4.1 API Endpoint

```
POST https://api-m.paypal.com/v1/payments/payouts
```

### 5.4.2 Request Structure

```json
{
  "sender_batch_header": {
    "sender_batch_id": "GJ_host123_1737093942",
    "email_subject": "Gap Junction Compute Earnings",
    "email_message": "Thank you for contributing compute resources."
  },
  "items": [{
    "recipient_type": "EMAIL",
    "receiver": "host_user@example.com",
    "amount": {
      "value": "50.00",
      "currency": "USD"
    },
    "note": "Gap Junction Host Payout",
    "sender_item_id": "GJ_host123_1737093942"
  }]
}
```

### 5.4.3 Security Constraints

| Constraint | Implementation |
|------------|----------------|
| No bank details | Store only `paypal_email` |
| No auto-payouts | User must click "Request Payout" |
| No FX handling | Always send USD; PayPal converts |
| Email masking | Logs show `ho***@example.com` |

---

## 5.5 Revenue Share Model

### 5.5.1 Distribution

$$
\text{Platform Fee} = \lfloor \text{Gross Amount} \times 0.15 \rfloor
$$

$$
\text{Host Earnings} = \text{Gross Amount} - \text{Platform Fee}
$$

| Actor | Share | Example ($1.00 task) |
|-------|-------|---------------------|
| Renter Pays | 100% | $1.00 |
| Platform | 15% | $0.15 |
| Host | 85% | $0.85 |

### 5.5.2 Platform Revenue Sources

1. **Compute Margin:** 15% of all task revenue
2. **Float:** Pre-paid credits held before spending
3. **Unused Credits:** Expired/abandoned balances (if ToS permits)

---

## 5.6 System Embodiment

The billing system is implemented in:

**Files:**
- `/controller/internal/billing/payouts.go` - Main service
- `/controller/internal/billing/ledger.go` - Data structures

**Key Constants:**
```go
const (
    MinPrepayAmountCents    = 2000  // $20.00
    MinPayoutThresholdCents = 5000  // $50.00
    FuelToCentsRatio        = 10_000_000_000
    PlatformTakeRateBps     = 1500  // 15.00%
)
```

---

## 5.7 Prior Art Differentiation

| System | Payment Model | Gap Junction Novelty |
|--------|---------------|----------------------|
| Golem | Per-task tokens | Pre-paid fiat credits |
| AWS Lambda | Per-invocation billing | Credit ledger prepay |
| Fiverr | Escrow per job | Aggregated payouts |
| Uber | Per-ride payment | Threshold-based payouts |

The combination of:
1. **Pre-paid credit ledger** ($20 minimum)
2. **Threshold-based payouts** ($50 minimum)
3. **Email-only routing** (no bank details)
4. **On-demand payout trigger** (no automatic)

...constitutes a novel fiat settlement system for distributed compute platforms.
