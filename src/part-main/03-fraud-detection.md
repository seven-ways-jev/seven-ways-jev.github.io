# Chapter 3: Financial Fraud Detection

## The Problem

Every payment processor faces the same tension: flag too little and fraudulent transactions go through; flag too much and legitimate customers get blocked and frustrated. Rules-based systems -- velocity checks, amount thresholds, country blocklists -- are fast but brittle. They miss novel fraud patterns and block legitimate transactions that happen to look unusual.

The deeper problem is that fraud detection is inherently a judgment call. A large jewelry purchase abroad is suspicious for most accounts and completely normal for others. A 4am pharmacy visit is unusual but not fraudulent. A significant electronics purchase at a known retailer in the customer's home city is probably fine, but it's above their typical monthly spend and worth a second look.

This chapter uses Jev to make a single binary decision for each incoming transaction: flag it for review, or pass it through? One call, two options, a confidence score. The confidence score is what makes this useful in production: a transaction flagged with high confidence goes straight to the fraud queue, while one flagged with lower confidence might warrant a softer intervention -- a one-time password or a confirmation SMS -- rather than an outright block.

## The Setup

We write 20 synthetic transactions by hand, spread across four bands, each carrying nine contextual signals:

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
                "be flagged for review or passed through? Consider the "
                "amount relative to typical spend, account age, location, "
                "time of day, velocity and any prior flags."
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

Each transaction was run once. Confidence scores vary a little between runs, so we describe the patterns here and leave the exact values to the notebook.

## What the Numbers Tell Us

### Clear cases are easy, edge cases are hard

Mean confidence by band tells the expected story at the extremes: near-maximum confidence for both clear fraud and clear legitimate. The drop into the ambiguous and edge case bands is substantial, and edge cases were no easier than ambiguous ones -- the unusual-but-legitimate patterns were as hard for Jev to call as the straightforward mixed-signal cases.

The bands hide some variation. Two of the five ambiguous transactions (the Paris restaurant and the jewelry purchase) were passed with near-certainty, while the other three were much less sure. And Jev flagged two of the five edge cases, including a large electronics purchase from a long-established account at a known retailer in the customer's home city. We designed that band as unusual but legitimate, so those two flags are arguably false positives. We didn't set expected answers, so the notebook can't score them, but they show where Jev's judgment and ours differ.

### Three examples worth looking at

**TX011** (large electronics purchase at a known retailer, home city, established account): passed with very low confidence. The merchant is legitimate and the location is right, but the amount is about 1.5 times this customer's typical monthly spend. Jev passed it but barely believed the decision.

**TX014** (ATM withdrawal, Las Vegas, late night, established account): also passed with very low confidence. Domestic, established account, not an extreme amount -- but a Las Vegas ATM late at night has a particular profile. Jev passed it and wasn't sure.

**TX017** (online gaming purchase at 3am, three transactions in the last hour): passed with one of the lowest confidences in the set, close to a coin flip. A late-night gaming purchase is normal for the category, but the hour and the velocity pull the other way and Jev couldn't separate them.

These are exactly the transactions that should trigger step-up authentication rather than a silent decision in either direction. Compare TX016, another electronics purchase from an established account at a known retailer, but at about 3.7 times the customer's monthly spend. Jev flagged it confidently, so the size of the amount relative to typical spend clearly matters.

### Category confidence reflects the tickets, not the category

The merchant category ranking is tempting to read as a property of each category, but most categories have only one or two transactions here, so it mostly reflects the individual tickets we wrote. With that caveat, the pattern looks like this:

- **Lowest confidence**: Online Gaming, Online Retail, Pharmacy, ATM Withdrawal -- tickets where context mattered far more than the category name
- **Middle**: Travel, Jewelry, Electronics -- amount and account history do the heavy lifting here
- **Highest confidence**: Restaurant, Gas Station, Grocery, Home Improvement, Wire Transfer, Gift Cards -- the last two are certain in the fraud direction, the others in the legitimate direction

Electronics is the most interesting category, appearing in clear fraud (new accounts, international, late night), an ambiguous case (TX011) and an edge case (TX016). The category name tells Jev almost nothing -- it needs the full context.

### Transactions for step-up authentication

The notebook uses a confidence threshold of 0.70. A handful of transactions fall below it, and they're mostly low-confidence passes: the uncertain ambiguous cases (TX011, TX013, TX014), the late-night gaming and pharmacy purchases (TX017, TX019) and the $12,000 international jewelry purchase (TX020). One flag also sits below the line, the $8,900 travel booking (TX018). The exact set shifts a little from run to run. Each of these is a case where a silent decision in either direction would be a guess, and each is a candidate for step-up authentication instead. The $12,000 purchase is the clearest example: Jev passed it, but with so little confidence that asking the customer to confirm is the safer route.

Flagged transactions carried higher confidence than passed ones on average, because the uncertain cases are mostly passes.

## Latency

One Jev call per transaction, each completing in a fraction of a second. That's fast enough to sit in the authorization path of a payment, though whether it fits depends on your latency budget and on how long you can wait for a step-up response.

## The Three-Tier Response

The confidence score enables a more nuanced response than binary flag-or-pass:

- **High confidence Flag** (above the threshold): route to fraud queue, hold transaction
- **High confidence Pass** (above the threshold): approve and process
- **Low confidence** (below the threshold): trigger step-up authentication, let the customer confirm

The threshold is a business decision, not a technical one. The right number depends on the cost of a false positive (a blocked legitimate customer) versus a false negative (a passed fraudulent transaction) in your specific context. What Jev provides is the infrastructure to make that trade-off explicit and tunable, rather than baking it invisibly into a set of rules.
