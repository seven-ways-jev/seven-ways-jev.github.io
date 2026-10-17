# Chapter 4: Content Moderation

## The problem

Platforms that host user-generated content face a decision for every post, comment and review: approve it, flag it for human review or remove it immediately. Get it wrong in either direction and the consequences are real. Miss a genuine threat and someone gets hurt. Remove legitimate content and you've silenced a user, damaged trust and potentially exposed the platform to legal challenge.

Rules-based moderation -- keyword blocklists, regex patterns, account age thresholds -- is fast but brittle. It misses context, catches satire and fails on novel patterns. Human moderation is accurate but doesn't scale.

This chapter uses Jev to make a three-option decision for each piece of content: approve it, flag it for human review, or remove it. The three-option structure is deliberate. A binary approve/remove decision forces a choice that shouldn't always be forced. The "Flag" option is meant to be where genuine uncertainty goes -- and a confidence score tells you how uncertain Jev actually is about each decision. We test whether it works that way.

## The setup

We write 20 posts by hand, spread across four bands, each carrying the content itself plus account signals and platform context that a real moderation system would have:

- **Clear violation** (5 posts): direct threats, spam with malicious links, hate speech, extortion
- **Clear acceptable** (5 posts): product reviews, technical questions, general discussion, support requests
- **Ambiguous** (5 posts): frustrated venting, strong political opinion, an angry product review, dark humor
- **Edge cases** (5 posts): satire that could be misread, a professional asking an unusual question, an angry cancellation request, a strong but legitimate political opinion

Each post carries five signals: platform (Social Media, Community Forum, Product Reviews, Customer Support), account age, prior violation count, verified status and the post content itself.

## One decision per post

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
                "decision should be applied? Consider the content itself, "
                "the platform it was posted on, the account's history and "
                "whether the account is verified."
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

Each post was run once for the table. The ten ambiguous and edge posts were also repeated five times, which we come back to below.

## What the numbers tell us

### Perfect at the extremes

All five clear violations were removed at maximum confidence. All five clear acceptable posts were approved at maximum confidence. No hesitation in either direction. The model has a strong, consistent signal at the extremes of the spectrum.

### The middle bands are mostly confident too

The middle bands didn't behave as labeled. Edge cases were no harder than ambiguous ones, and several of the ambiguous posts (the landlord venting, the Monday-morning joke, the ER dark humor) were approved with near-certainty. Only a few posts in either band really were uncertain.

### Jev rarely uses the middle option

The most notable finding is how rarely the Flag option was used. Across the 10 ambiguous and edge case posts, Flag appeared for exactly one post -- P014, the angry product review threatening reputation damage -- and for that post every time we repeated the run. Every other middle-band post got the same decision on every repeat: Approve. Even the single flag isn't guaranteed, though. In one earlier run Jev removed P014 instead.

In this setup, Jev doesn't naturally spread decisions across all three options. The decisions were stable from run to run and the uncertainty surfaced in the confidence score instead of in the choice of the middle option. The few posts that fell below the confidence threshold were mostly approvals Jev wasn't sure about. Those low-confidence signals are more useful than the Flag option itself.

### The posts worth reviewing

Three posts fall below the threshold and a fourth sits right on it.

**P012** (strong political opinion about government failures): legitimate political expression but heated enough to give Jev pause. Low confidence signals "probably fine but worth a look."

**P014** (angry product review calling the company "criminals" and threatening to damage their reputation): the one post Jev flagged. It's genuinely borderline -- the language is aggressive but the underlying complaint is a legitimate consumer grievance. Flagging rather than removing looks like the right call.

**P018** (customer threatening chargebacks and negative reviews unless their issue is resolved): on a customer support platform this reads as a frustrated customer at the end of their patience, not a bad actor. Low confidence is exactly the honest signal here.

**P019** (a pharmacist asking about their professional obligation when customers request concerning medication combinations): a legitimate professional question on a sensitive topic. Its confidence hovered around the threshold from run to run, so it moves in and out of the review list.

### Platform confidence

Customer Support had the lowest mean confidence, driven by P018. Social Media had the highest, because every Social Media post in the set happened to be one Jev was sure about, including the ambiguous and edge ones. With only a few posts per platform, and platform tangled up with band, we can't say the platform itself changed the decisions. A fair test would put the same text on different platforms, which we haven't done.

### Decision distribution

The Flag option was used once and with low confidence. In practice, the low-confidence Approve decisions are functionally equivalent to flags -- they should join the human review queue regardless of which option Jev chose.

## Latency

One Jev call per post, each completing in a fraction of a second, so all twenty posts took about five seconds.

## The practical takeaway

The three-option structure is correct in theory but the confidence score is the more informative signal in practice. Rather than routing purely on the decision (Approve / Flag / Remove), a production moderation system should route on decision and confidence together:

- **Remove above the confidence threshold**: remove immediately
- **Approve above the confidence threshold**: publish immediately
- **Anything below the confidence threshold, regardless of decision**: route to human review

This reframes Flag not as a specific outcome but as a proxy for "confidence below threshold" -- and the confidence score makes that threshold explicit and tunable.

One caution: we didn't check the decisions against expected answers, so the scores show where Jev is unsure, not how often it's right.
