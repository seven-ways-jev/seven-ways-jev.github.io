# Chapter 3: Financial Fraud Detection

## The Problem

Every payment processor faces the same tension: flag too little and fraudulent transactions go through; flag too much and legitimate customers get blocked and frustrated. Rules-based systems -- velocity checks, amount thresholds, country blocklists -- are fast but brittle. They miss novel fraud patterns and block legitimate transactions that happen to look unusual.

The deeper problem is that fraud detection is inherently a judgment call. A large jewelry purchase abroad is suspicious for most accounts and completely normal for others. A 4am pharmacy visit is unusual but not fraudulent. A significant electronics purchase at a known retailer in the customer's home city is probably fine, but it's well above their typical monthly spend and worth a second look.

This chapter uses Jev to make a single binary decision for each incoming transaction: flag it for review, or pass it through? One call, two options, a calibrated confidence score. The confidence score is what makes this useful in production: a transaction flagged with high confidence goes straight to the fraud queue, while one flagged with lower confidence might warrant a softer intervention -- a one-time password or a confirmation SMS -- rather than an outright block.

## The Setup

We generate 20 synthetic transactions across four bands, each carrying nine contextual signals:

- Amount in USD
- Merchant category and name
- Location (home city, domestic, international)
- Time of day
- Account age in days
- Number of previous flags on the account
- Transactions in the last hour (velocity)
- Typical monthly spend

The four bands are designed to test the full range of difficulty: clear fraud (multiple stacked risk signals), clear legitimate (established account, normal behavior), ambiguous (one or two flags but not conclusive) and edge cases (unusual but potentially legitimate patterns).

## One Decision per Transaction

Each transaction goes through a single `Choice` call. All nine signals go into the state so Jev can weigh them together rather than applying rules sequentially.

```python
response = client.system_one(
    state={
        "amount_usd":                tx["amount_usd"],
        "merchant_category":         tx["merchant_category"],
        "merchant_name":             tx["merchant_name"],
        "location":                  tx["location"],
        "time_of_day":               tx["time_of_day"],
        "account_age_days":          tx["account_age_days"],
        "previous_flags":            tx["previous_flags"],
        "transactions_last_hour":    tx["transactions_last_hour"],
        "typical_monthly_spend_usd": tx["typical_monthly_spend_usd"],
    },
    questions={
        "decision": Choice(
            instructions=(
                "Based on all available signals, should this transaction "
                "be flagged for review or passed through?"
            ),
            criteria=FRAUD_DECISION,
        )
    },
)
```

## Results

| ID | Band | Amount | Category | Decision |
|---|---|---|---|---|
| TX001 | Clear fraud | $4,200 | Electronics | Flag |
| TX002 | Clear fraud | $9,800 | Wire Transfer | Flag |
| TX003 | Clear fraud | $650 | ATM Withdrawal | Flag |
| TX004 | Clear fraud | $1,500 | Gift Cards | Flag |
| TX005 | Clear fraud | $3,100 | Electronics | Flag |
| TX006 | Clear legit | $87 | Grocery | Pass |
| TX007 | Clear legit | $42 | Restaurant | Pass |
| TX008 | Clear legit | $320 | Travel | Pass |
| TX009 | Clear legit | $55 | Gas Station | Pass |
| TX010 | Clear legit | $1,200 | Home Improvement | Pass |
| TX011 | Ambiguous | $2,800 | Electronics | Pass |
| TX012 | Ambiguous | $180 | Restaurant | Pass |
| TX013 | Ambiguous | $95 | Online Retail | Pass |
| TX014 | Ambiguous | $450 | ATM Withdrawal | Pass |
| TX015 | Ambiguous | $600 | Jewelry | Pass |
| TX016 | Edge | $5,500 | Electronics | Flag |
| TX017 | Edge | $230 | Online Gaming | Pass |
| TX018 | Edge | $8,900 | Travel | Flag |
| TX019 | Edge | $75 | Pharmacy | Pass |
| TX020 | Edge | $12,000 | Jewelry | Pass |

## What the Numbers Tell Us

### Clear cases are easy, edge cases are hard

Mean confidence by band tells the expected story at the extremes: maximum confidence for both clear fraud and clear legitimate. The drop into the ambiguous and edge case bands is substantial and edge cases actually produced lower mean confidence than ambiguous ones -- the unusual-but-legitimate patterns were harder for Jev to call than the straightforward mixed-signal cases.

### The two most interesting results

**TX011** (large electronics purchase at a known retailer, home city, established account): passed with very low confidence. The merchant is legitimate and the location is right, but the amount is well above this customer's typical monthly spend. Jev passed it but barely believed the decision.

**TX014** (ATM withdrawal, Las Vegas, late night, established account): also passed with very low confidence. Domestic, established account, not an extreme amount -- but a Las Vegas ATM late at night has a particular profile. Jev passed it and wasn't sure.

Both of these are exactly the transactions that should trigger step-up authentication rather than a silent decision in either direction.

### Category confidence reveals inherent uncertainty

The merchant category confidence ranking shows which categories carry inherent uncertainty regardless of other signals:

- **Highest uncertainty**: Pharmacy, ATM Withdrawal, Online Gaming, Online Retail -- categories where context matters far more than the category name
- **Middle**: Travel, Jewelry, Electronics -- amount and account history do the heavy lifting here
- **Lowest uncertainty**: Restaurant, Gas Station, Grocery, Home Improvement, Wire Transfer, Gift Cards -- the last two are certain in the fraud direction, the others in the legitimate direction

Electronics is the most interesting category across multiple transactions, appearing in both clear fraud (new accounts, international, late night) and edge cases (established account, home city, known retailer). The category name tells Jev almost nothing -- it needs the full context.

### Transactions flagged for step-up authentication

Below the confidence threshold, several transactions fall into the uncertain zone and should trigger a softer intervention rather than an automatic decision. These are the genuinely uncertain cases that a rules-based system would have decided silently.

## Latency

One Jev call per transaction, completing in well under a second each. At this latency, a production fraud detection system could assess transactions in real time before they complete, with enough headroom to trigger step-up authentication and wait for the customer's response before the payment times out.

## The Three-Tier Response

The confidence score enables a more nuanced response than binary flag-or-pass:

- **High confidence Flag** (above the threshold): route to fraud queue, hold transaction
- **High confidence Pass** (above the threshold): approve and process
- **Low confidence** (below the threshold): trigger step-up authentication, let the customer confirm

The threshold is a business decision, not a technical one. The right number depends on the cost of a false positive (a blocked legitimate customer) versus a false negative (a passed fraudulent transaction) in your specific context. What Jev provides is the infrastructure to make that trade-off explicit and tunable, rather than baking it invisibly into a set of rules.
