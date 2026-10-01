# Chapter 2: Customer Support Triage

## The Problem

Every support operation faces the same challenge: an incoming ticket needs to reach the right team at the right priority and it needs to happen fast. Get it wrong and a critical production outage sits in the general queue. A billing dispute gets escalated to the engineering team. A security researcher's responsible disclosure report sits unanswered for three days.

Rules-based routing -- keyword matching, tier lookups, category flags -- works until the edge cases arrive. A Gold customer whose "quick question" turns out to be a production outage. A billing dispute that's actually an account security issue. A Free-tier user whose problem, if ignored, will end up in a public forum.

This chapter uses Jev to make two sequential decisions for each incoming ticket: which team should handle it and at what priority? Each decision returns a calibrated confidence score alongside the answer. That confidence score is the key difference from a rules-based router: instead of silently routing every ticket with equal certainty, Jev flags the ones it's unsure about so a human can review them before they go anywhere.

## The Setup

No external dataset is needed. We generate 20 synthetic tickets spread across four bands designed to test the model across a range of difficulty:

- **Clear high-priority**: unambiguous critical or urgent issues (API outage, security breach, large overbilling)
- **Clear low-priority**: unambiguous routine requests (how-to questions, invoice copies, feature requests)
- **Ambiguous**: mixed signals where the right routing is genuinely unclear (slow dashboard, integration not syncing, login error with a deadline)
- **Edge cases**: unusual combinations that test the boundaries (responsible disclosure report, churn risk from an Enterprise customer, upgrade blocked by a broken billing page)

Each ticket has a customer tier (Free, Pro, Enterprise), an issue category, a subject line and a description written the way a real customer might write it.

We route to four teams: Billing, Technical, Account Management and General Support. Priority levels are Critical, High, Medium and Low.

## Two Decisions per Ticket

Each ticket goes through two sequential `Choice` calls. The team decision uses the full ticket context. The priority decision adds the team assignment to the state so Jev can factor in who will be handling it before deciding how urgently they need to act.

```python
# Decision 1: Team
team_response = client.system_one(
    state=state,
    questions={
        "team": Choice(
            instructions="Which support team should handle this ticket?",
            criteria=TEAMS,
        )
    },
)

# Decision 2: Priority (state now includes assigned team)
state["assigned_team"] = team_response.choices["team"].choice
priority_response = client.system_one(
    state=state,
    questions={
        "priority": Choice(
            instructions="What priority level should this ticket receive?",
            criteria=PRIORITIES,
        )
    },
)
```

## Results

| ID | Band | Team | Priority |
|---|---|---|---|
| T001 | Clear high | Technical | Critical |
| T002 | Clear high | Billing | High |
| T003 | Clear high | Technical | Critical |
| T004 | Clear high | Account Management | High |
| T005 | Clear high | Technical | Critical |
| T006 | Clear low | General Support | Low |
| T007 | Clear low | Billing | Low |
| T008 | Clear low | General Support | Low |
| T009 | Clear low | General Support | Low |
| T010 | Clear low | Billing | Low |
| T011 | Ambiguous | Technical | Medium |
| T012 | Ambiguous | Billing | High |
| T013 | Ambiguous | General Support | Medium |
| T014 | Ambiguous | Technical | High |
| T015 | Ambiguous | Technical | High |
| T016 | Edge | Technical | High |
| T017 | Edge | Account Management | High |
| T018 | Edge | Technical | High |
| T019 | Edge | Technical | High |
| T020 | Edge | General Support | Critical |

## What the Numbers Tell Us

### Team routing is more certain than priority

Across all 20 tickets, Jev was consistently more confident about which team should handle a ticket than about what priority it deserved. That gap is meaningful: knowing where a ticket belongs is often clearer than knowing how urgently it needs to be handled.

### Clear tickets aren't always easier

The confidence-by-band chart shows an unexpected result: clear high-priority tickets had lower mean priority confidence than clear low-priority ones. The reason is that "Low" is easy to assign with certainty -- a routine how-to question is unambiguously low priority. But "Critical vs High" is a harder call even for obvious tickets. The large overbilling ticket is a good example: it's not a service outage, but it's a serious financial error. Jev's hesitation there is correct.

### Tier is a weak signal

Mean confidence barely varies across customer tiers. Enterprise tickets are slightly easier to route, probably because Enterprise customers write more detailed descriptions. But the difference is small. A rules-based router would give Enterprise tickets automatic priority treatment based on tier alone. Jev is making decisions based on what the ticket actually says.

### The three tickets worth flagging

At the confidence threshold, three tickets fall below the line:

**T016** (security researcher's responsible disclosure): high team confidence but low priority confidence. Jev knows it's Technical but genuinely doesn't know whether a responsible disclosure report is High or Critical priority. That's an honest answer -- it depends on company policy, not just the ticket content.

**T018** (Pro customer trying to upgrade, billing page broken): low team confidence. It's simultaneously a billing issue and a technical issue -- a broken UI on a payment page. Jev can't decide which team owns this and it shouldn't pretend otherwise.

**T020** (Free user worried about unauthorized account access): low confidence on both team and priority. Routed to General Support at Critical priority. Arguably should go to Technical since it's potentially a security incident. Both the routing and the urgency are uncertain and the low confidence says so.

These are exactly the tickets a human reviewer should see before they go anywhere.

## Latency

Two Jev calls per ticket, running sequentially. Twenty tickets completed in well under a minute. In a production system the two calls could run in parallel for tickets where team and priority are independent, cutting total time roughly in half. Or the priority call could be deferred -- route the ticket immediately on team, then assign priority asynchronously before the agent picks it up.

## The Confidence Threshold

The threshold is a business decision, not a technical one. Lower it and more tickets get human review -- slower but fewer routing errors. Raise it and more tickets route automatically -- faster but more misrouted ones. The right number depends on the cost of a wrong routing decision in your operation.

What Jev provides is the infrastructure to make that trade-off explicit. A rules-based router routes every ticket with implicit certainty. Jev routes most tickets automatically and surfaces the ones where certainty runs out.
