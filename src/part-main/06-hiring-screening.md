# Chapter 6: Hiring screening

## Important disclaimer

This chapter is illustrative only. Jev is assessing structured summaries against structured requirements -- it is not making final hiring decisions and should not be used as one. Any real implementation of AI-assisted hiring screening requires human oversight, regular bias auditing and compliance with applicable employment law. The candidates and job description in this chapter are entirely fictional.

## The problem

Recruiting teams face a familiar bottleneck: a role opens, applications arrive faster than anyone can read them and the risk of a bad screen in either direction is real. Miss a strong candidate and they accept another offer. Advance a poor fit and you waste an interviewer's time and the candidate's.

Rules-based screening -- keyword matching on required skills, minimum years of experience, degree requirements -- is fast but crude. It filters on the presence of words rather than the quality of fit. A candidate who writes "built distributed systems" passes; one who writes "designed and shipped a high-throughput event pipeline" might not, even if the latter is the stronger hire.

This chapter uses Jev to make a three-option screening decision for each candidate: advance them to interview, flag them for human review, or decline. One `Choice` call per candidate, structured CV summary against structured job requirements, a confidence score that tells you how certain the decision is.

## The setup

One job description -- Senior Backend Engineer, Platform Engineering -- and 20 hand-written candidate CV summaries across four bands:

- **Strong match** (5 candidates): clearly meets all requirements -- right experience, right stack, right seniority
- **Clear mismatch** (5 candidates): clearly doesn't meet requirements -- wrong field, too junior, missing core skills
- **Borderline** (5 candidates): partially meets requirements -- right stack but too junior, right seniority but wrong stack, adjacent field
- **Edge cases** (5 candidates): unusual profiles that a keyword filter would handle badly -- overqualified, employment gap, vague CV with strong GitHub, recent stack mismatch, career changer without a CS degree

Some summaries state their own ambiguities in words (for example, that a gap "may indicate recency concerns"), as a recruiter's screening notes might. Real CVs wouldn't arrive pre-assessed, so part of what Jev does here is read the cue we wrote. Treat the results as an illustration.

## One decision per candidate

One `Choice` call per candidate. The job requirements go into the state alongside the candidate's structured CV summary.

```python
response = client.system_one(
    state={
        "role_title":        job["title"],
        "seniority_level":   job["level"],
        "required_skills":   job["required_skills"],
        "preferred_skills":  job["preferred_skills"],
        "not_suitable_if":   job["not_suitable"],
        "candidate_summary": candidate["summary"],
        "years_experience":  candidate["years_experience"],
        "current_title":     candidate["current_title"],
    },
    questions={
        "decision": Choice(
            instructions=(
                "Based on the candidate's experience and the role requirements, "
                "should this candidate be advanced to interview, flagged for "
                "human review, or declined? Assess fit on required skills, "
                "seniority level and relevant experience."
            ),
            criteria=SCREENING_DECISION,
        )
    },
)
```

## Results

| ID | Name | Band | Decision |
|---|---|---|---|
| C001 | Alex Rivera | Strong match | Advance |
| C002 | Jordan Kim | Strong match | Advance |
| C003 | Sam Okonkwo | Strong match | Advance |
| C004 | Maya Patel | Strong match | Advance |
| C005 | Chris Nakamura | Strong match | Advance |
| C006 | Taylor Brooks | Clear mismatch | Decline |
| C007 | Morgan Ellis | Clear mismatch | Decline |
| C008 | Jamie Foster | Clear mismatch | Decline |
| C009 | Riley Chen | Clear mismatch | Decline |
| C010 | Casey Wong | Clear mismatch | Decline |
| C011 | Dana Osei | Borderline | Decline |
| C012 | Avery Santos | Borderline | Review |
| C013 | Quinn Murphy | Borderline | Decline |
| C014 | Skyler Park | Borderline | Review |
| C015 | Drew Hassan | Borderline | Review |
| C016 | Frankie Diaz | Edge | Review |
| C017 | Reese Andersen | Edge | Review |
| C018 | Blair Kim | Edge | Review |
| C019 | Robin Castillo | Edge | Decline |
| C020 | Sage Thompson | Edge | Review |

Each candidate was run once for this table. The ten borderline and edge candidates were also repeated five times in each of two option orders, which we come back to below.

## What the numbers tell us

### Review used as a middle ground

Unlike the content moderation chapter, where Flag appeared for just one post, Review appeared for seven candidates -- three borderline and four edge cases. The repeat runs make this a firm result: every one of the ten borderline and edge candidates got the same decision on every repeat, in both option orders.

We shouldn't credit this too confidently to the nature of hiring. The Review option is described as a profile that "warrants closer human assessment", which is a more specific invitation than the Flag option in chapter 4, and the question itself names human review. Several summaries also say outright that the situation is ambiguous. Part of the difference is how the options and the inputs were written.

The decision distribution is Advance for the five strong matches, Decline for the five clear mismatches and a spread of Review and Decline across the harder cases. On average Advance and Decline were more confident than Review, which sits in the middle.

### Confidence drops with difficulty

Strong matches and clear mismatches were both near maximum confidence. Borderline cases were lower and edge cases lowest on average -- genuinely harder calls.

### Individual results worth a look

**C005 (Chris Nakamura)**: the strong match with noticeably lower confidence than the rest. Five years is exactly at the minimum threshold and the CV notes no Kafka experience. Jev advanced him but with less confidence than for the stronger candidates, which fits the CV.

**C012 (Avery Santos)**: a Java engineer with seven years of experience and strong distributed systems background -- but no Python or Go. The lowest confidence in the borderline band. Experience level and systems depth are right, but the stack mismatch is real.

**C015 (Drew Hassan)**: sole backend engineer at a startup for five years -- strong ownership signals but no peer review, team collaboration or mentoring context. Jev put this in Review with high confidence, flagging that human judgment is needed rather than an automatic advance or decline.

**C016 (Frankie Diaz)**: overqualified Principal Engineer with twelve years of experience applying for a Senior role. Jev sent this to Review rather than advancing or declining. Is this a deliberate step back? A management-to-IC transition? Those questions need a conversation -- and our summary raises them outright, so Jev had the cue.

**C018 (Blair Kim)**: vague CV language with strong GitHub contributions suggesting better skills than the CV conveys. Jev sent this to Review with very high confidence -- the highest of any Review decision. The summary says outright that it's unclear whether the CV reflects limited experience or poor writing, which is the signal Jev picked up.

**C019 (Robin Castillo)**: seven years experience but the most recent three years in PHP at a web agency. Jev declined with moderate confidence -- acknowledging the earlier Python experience but deciding the recent stack distance was too significant. Lower confidence than the clear mismatch declines, which fits.

**C020 (Sage Thompson)**: career changer with no CS degree, self-taught, six years of shipped backend work. Jev put this in Review rather than declining, in keeping with the point that non-traditional backgrounds deserve human assessment rather than automatic rejection.

### Option order and repeat runs

Across five repeats in each option order, every borderline and edge candidate got the same decision every time. Confidence moved a little, and by more for a few candidates: for C016 and C017 it rose when the options were reversed, to close to or across the review threshold used below. So the decisions looked robust to order here, but the confidence number is less so, which matters when a fixed threshold decides who sees a human. It's the same lesson as in the supply chain chapter.

## Latency

One Jev call per candidate, each completing in a fraction of a second, so all twenty candidates were screened in about five seconds.

## Fairness

A few design choices matter for fairness. We don't pass candidate names to Jev; it sees only the job requirements, the current title, the years of experience and the summary. And some of the signals in our edge cases, such as an employment gap or being overqualified, are commonly associated with caregiving, disability and age. Here they led to Review, not Decline, which is the safer outcome, but a real system should make sure such signals never count against a candidate and should audit its outcomes across groups.

## The right architecture

Jev screening handles the easy cases automatically and surfaces the hard ones. In the notebook a candidate goes to a human recruiter if Jev chose Review or if confidence falls below a threshold of 0.70. With that rule, nine of the twenty candidates went to a human in our run: all seven Reviews and two low-confidence Declines. One of those, C013, sits right on the line and moves in and out of the human queue between runs. High-confidence Advance decisions move to interview scheduling and the rest are processed automatically.

Automatic declines deserve the most care. A rejection has the most consequence for the candidate, and the automatic Declines included borderline candidates such as C011, who is below the seniority bar but described as growing quickly. A team might reasonably require a human to review every borderline decline, or set a higher threshold for declines than for advances.

This is not a replacement for human judgment. It's a way to ensure that human judgment is applied where it matters most, rather than being consumed by decisions that are objectively straightforward.

One caution: we didn't check any decision against an expected answer, so the scores show where Jev is unsure, not how often it's right.
