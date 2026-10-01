# Chapter 4: Content Moderation

## The Problem

Platforms that host user-generated content face a decision for every post, comment and review: approve it, flag it for human review or remove it immediately. Get it wrong in either direction and the consequences are real. Miss a genuine threat and someone gets hurt. Remove legitimate content and you've silenced a user, damaged trust and potentially exposed the platform to legal challenge.

Rules-based moderation -- keyword blocklists, regex patterns, account age thresholds -- is fast but brittle. It misses context, catches satire and fails on novel patterns. Human moderation is accurate but doesn't scale.

This chapter uses Jev to make a three-option decision for each piece of content: approve it, flag it for human review, or remove it. The three-option structure is deliberate. A binary approve/remove decision forces a choice that shouldn't always be forced. The "Flag" option is where genuine uncertainty belongs -- and a calibrated confidence score tells you how uncertain Jev actually is about each decision.

## The Setup

We generate 20 posts across four bands, each carrying the content itself plus account signals and platform context that a real moderation system would have:

- **Clear violation** (5 posts): direct threats, spam with malicious links, hate speech, extortion
- **Clear acceptable** (5 posts): product reviews, technical questions, general discussion, support requests
- **Ambiguous** (5 posts): frustrated venting, strong political opinion, an angry product review, dark humor
- **Edge cases** (5 posts): satire that could be misread, a professional asking an unusual question, an angry cancellation request, a strong but legitimate political opinion

Each post carries five signals: platform (Social Media, Community Forum, Product Reviews, Customer Support), account age, prior violation count, verified status and the post content itself.

## One Decision per Post

One `Choice` call per post, three options: Approve, Flag or Remove.

```python
response = client.system_one(
    state={
        "platform":         post["platform"],
        "account_age_days": post["account_age_days"],
        "prior_violations": post["prior_violations"],
        "verified_account": post["verified"],
        "content":          post["content"],
    },
    questions={
        "decision": Choice(
            instructions=(
                "Based on the content and account context, what moderation "
                "decision should be applied?"
            ),
            criteria=MODERATION_DECISION,
        )
    },
)
```

## Results

| ID | Band | Decision |
|---|---|---|
| P001 | Clear violation | Remove |
| P002 | Clear violation | Remove |
| P003 | Clear violation | Remove |
| P004 | Clear violation | Remove |
| P005 | Clear violation | Remove |
| P006 | Clear acceptable | Approve |
| P007 | Clear acceptable | Approve |
| P008 | Clear acceptable | Approve |
| P009 | Clear acceptable | Approve |
| P010 | Clear acceptable | Approve |
| P011 | Ambiguous | Approve |
| P012 | Ambiguous | Approve |
| P013 | Ambiguous | Approve |
| P014 | Ambiguous | Flag |
| P015 | Ambiguous | Approve |
| P016 | Edge | Approve |
| P017 | Edge | Approve |
| P018 | Edge | Approve |
| P019 | Edge | Approve |
| P020 | Edge | Approve |

## What the Numbers Tell Us

### Perfect at the extremes

All five clear violations were removed at maximum confidence. All five clear acceptable posts were approved at maximum confidence. No hesitation in either direction. The model has a strong, consistent signal at the extremes of the spectrum.

### Jev prefers binary decisions

The most notable finding is how rarely the Flag option was used. Across 10 ambiguous and edge case posts, Flag appeared exactly once -- P014, the angry product review threatening reputation damage. Jev leaned heavily toward Approve across the middle bands.

This tells us something important about how Jev handles a three-option choice: it doesn't naturally distribute decisions across all three options. It tends toward confident binary decisions and the confidence score is where the uncertainty actually surfaces rather than the choice of the middle option. The posts that fell below the confidence threshold are all approved or flagged decisions Jev wasn't sure about. Those low-confidence signals are more useful than the Flag option itself.

### The three posts worth reviewing

**P012** (strong political opinion about government failures): legitimate political expression but heated enough to give Jev pause. Low confidence signals "probably fine but worth a look."

**P014** (angry product review calling the company "criminals" and threatening to damage their reputation): the one post Jev flagged. It's genuinely borderline -- the language is aggressive but the underlying complaint is a legitimate consumer grievance. Jev got this right by flagging rather than removing.

**P018** (customer threatening chargebacks and negative reviews unless their issue is resolved): on a customer support platform this reads as a frustrated customer at the end of their patience, not a bad actor. Low confidence is exactly the honest signal here.

### Platform confidence

Customer Support had the lowest mean confidence, driven by P018. Social Media had the highest, because it contained the clearest examples at both extremes. Community Forum and Product Reviews sat in the middle. The platform context influenced decisions at the margins -- the same angry language reads differently on a customer support channel than on social media.

### Decision distribution

The Flag option was used once and with low confidence. In practice, the low-confidence Approve decisions are functionally equivalent to flags -- they should join the human review queue regardless of which option Jev chose.

## Latency

One Jev call per post, completing in well under a second each. Twenty posts processed in well under a minute.

## The Practical Takeaway

The three-option structure is correct in theory but the confidence score is the more reliable signal in practice. Rather than routing purely on the decision (Approve / Flag / Remove), a production moderation system should route on decision and confidence together:

- **Remove at any confidence**: act immediately
- **Approve above the threshold**: publish immediately
- **Anything below the threshold**: route to human review regardless of decision

This reframes Flag not as a specific outcome but as a proxy for "confidence below threshold" -- and the confidence score makes that threshold explicit and tunable.
