# Chapter 2: Customer Support Triage

## The problem

Every support operation faces the same challenge: an incoming ticket needs to reach the right team at the right priority and it needs to happen fast. Get it wrong and a critical production outage sits in the general queue. A billing dispute gets escalated to the engineering team. A security researcher's responsible disclosure report sits unanswered for three days.

Rules-based routing -- keyword matching, tier lookups, category flags -- works until the edge cases arrive. An Enterprise customer whose "quick question" turns out to be a production outage. A billing dispute that's actually an account security issue. A Free-tier user whose problem, if ignored, will end up in a public forum.

This chapter uses Jev to make two sequential decisions for each incoming ticket: which team should handle it and at what priority? Each decision returns a confidence score alongside the answer. That score is the key difference from a rules-based router: instead of silently routing every ticket with equal certainty, the system can use Jev's confidence to flag the ones it's unsure about so a human can review them before they go anywhere.

## The setup

No external dataset is needed. We write 20 synthetic tickets by hand, spread across four bands designed to test the model across a range of difficulty:

- **Clear high-priority**: unambiguous critical or urgent issues (API outage, security breach, large overbilling)
- **Clear low-priority**: unambiguous routine requests (how-to questions, invoice copies, feature requests)
- **Ambiguous**: mixed signals where the right routing is genuinely unclear (slow dashboard, integration not syncing, login error with a deadline)
- **Edge cases**: unusual combinations that test the boundaries (responsible disclosure report, churn risk from an Enterprise customer, upgrade blocked by a broken billing page)

Each ticket has a customer tier (Free, Pro, Enterprise), an issue category, a subject line and a description written the way a real customer might write it.

We route to four teams: Billing, Technical, Account Management and General Support. Priority levels are Critical, High, Medium and Low.

## Two decisions per ticket

Each ticket goes through two sequential `Choice` calls. The team decision uses the full ticket context. The priority decision adds the team assignment to the state so Jev can factor in who will be handling it before deciding how urgently they need to act.

```python
# Decision 1: Team (instructions shortened here; the notebook has the full text)
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

Each ticket was run once. Confidence scores vary a little between runs, so we describe the patterns here and leave the exact values to the notebook.

## What the numbers tell us

### Team routing is a little more certain than priority

On average Jev was more confident about which team should handle a ticket than about what priority it deserved. That isn't true of every ticket: on a handful, priority confidence was equal or higher. The gap is still meaningful: knowing where a ticket belongs is often clearer than knowing how urgently it needs to be handled.

### Clear tickets aren't always easier

The confidence-by-band chart shows an unexpected result: clear high-priority tickets had lower mean priority confidence than clear low-priority ones. The reason is that "Low" is easy to assign with certainty -- a routine how-to question is unambiguously low priority. But "Critical vs High" is a harder call even for obvious tickets. The large overbilling ticket is a good example: it's not a service outage, but it's a serious financial error. Jev's hesitation there is understandable.

The ambiguous tickets make the same point. Their priority confidence was at least as high as that of the clear high-priority tickets and their team confidence was just as high. Only the edge cases were really harder.

### Tier makes little difference to confidence

Mean confidence varies little across customer tiers and not consistently in one direction. Enterprise tickets tended to be easier to route to a team, but that advantage didn't carry over to priority. With six or seven tickets per tier and descriptions we wrote ourselves, these differences are too small to read much into. Tier is part of the state Jev sees, so this comparison can't tell us how much it drives the decisions.

### The tickets worth flagging

At the 0.60 threshold, a handful of tickets fall below the line. The exact set shifts a little from run to run, but these four did in our runs:

**T002** (Enterprise customer charged $48,000 instead of $4,800): high team confidence but low priority confidence. It isn't an outage, but it's a serious financial error, so High versus Critical is a real judgment call.

**T003** (Pro customer whose data export is broken): sure about the team, unsure whether Critical or High. The feature is completely broken but it's a reporting task with a two-hour deadline, so it's a close call.

**T016** (security researcher's responsible disclosure): high team confidence but low priority confidence. Jev knows it's Technical but genuinely doesn't know whether a responsible disclosure report is High or Critical priority. That's an honest answer -- it depends on company policy, not just the ticket content.

**T018** (Pro customer trying to upgrade, billing page broken): low team confidence. It's simultaneously a billing issue and a technical issue -- a broken UI on a payment page. Jev can't decide which team owns this and it shouldn't pretend otherwise.

A few more sit close to the line and move in and out of the review list between runs. **T007** (Free user asking about storage limits on the free plan) went to Billing but could as easily go to General Support, and the team confidence reflects that. **T020** (Free user worried about unauthorized account access) went to General Support at Critical priority with both confidences near the threshold. It arguably should go to Technical since it's potentially a security incident. Which side of the line tickets like these land on is exactly why the threshold is a business decision.

These are the tickets a human reviewer should see before they go anywhere.

## Latency

Two Jev calls per ticket, running sequentially. Each ticket took roughly half a second, so all twenty completed in about ten seconds. In a production system the two calls could run in parallel for tickets where team and priority are independent, cutting total time roughly in half. Or the priority call could be deferred -- route the ticket immediately on team, then assign priority asynchronously before the agent picks it up.

## The confidence threshold

The threshold is a business decision, not a technical one. Lower it and more tickets get human review -- slower but fewer routing errors. Raise it and more tickets route automatically -- faster but more misrouted ones. The right number depends on the cost of a wrong routing decision in your operation.

One caution: we didn't check the routing against expected answers and each ticket ran once. The scores show where Jev is unsure, not how often it's right or how stable its answers are across repeated runs.

What Jev provides is the infrastructure to make that trade-off explicit. A rules-based router routes every ticket with implicit certainty. Jev routes most tickets automatically and surfaces the ones where certainty runs out.
