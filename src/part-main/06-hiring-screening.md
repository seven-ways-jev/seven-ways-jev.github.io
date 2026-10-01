# Chapter 6: Hiring Screening

## Important Disclaimer

This chapter is illustrative only. Jev is assessing structured summaries against structured requirements -- it is not making final hiring decisions and should not be used as one. Any real implementation of AI-assisted hiring screening requires human oversight, regular bias auditing and compliance with applicable employment law. The candidates and job description in this chapter are entirely fictional.

## The Problem

Recruiting teams face a familiar bottleneck: a role opens, applications arrive faster than anyone can read them and the risk of a bad screen in either direction is real. Miss a strong candidate and they accept another offer. Advance a poor fit and you waste an interviewer's time and the candidate's.

Rules-based screening -- keyword matching on required skills, minimum years of experience, degree requirements -- is fast but crude. It filters on the presence of words rather than the quality of fit. A candidate who writes "built distributed systems" passes; one who writes "designed and shipped a high-throughput event pipeline" might not, even if the latter is the stronger hire.

This chapter uses Jev to make a three-option screening decision for each candidate: advance them to interview, flag them for human review, or decline. One `Choice` call per candidate, structured CV summary against structured job requirements, a calibrated confidence score that tells you how certain the decision is.

## The Setup

One job description -- Senior Backend Engineer, Platform Engineering -- and 20 candidate CV summaries across four bands:

- **Strong match** (5 candidates): clearly meets all requirements -- right experience, right stack, right seniority
- **Clear mismatch** (5 candidates): clearly doesn't meet requirements -- wrong field, too junior, missing core skills
- **Borderline** (5 candidates): partially meets requirements -- right stack but too junior, right seniority but wrong stack, adjacent field
- **Edge cases** (5 candidates): unusual profiles that a keyword filter would handle badly -- overqualified, employment gap, vague CV with strong GitHub, recent stack mismatch, career changer without a CS degree

## One Decision per Candidate

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
                "human review, or declined?"
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

## What the Numbers Tell Us

### Review used as a genuine middle ground

Unlike the content moderation chapter where Flag appeared just once, Review appeared seven times -- across three borderline candidates and four edge cases. Hiring decisions are inherently more nuanced than content moderation and Jev reflects that: it used the middle option where the middle option was warranted.

The decision distribution shows a clean pattern: Advance for the five strong matches, Decline for the five clear mismatches and a genuine spread of Review and Decline across the harder cases. Mean confidence by decision tells the right story too -- Advance and Decline are more confident decisions than Review, which sits in the middle as it should.

### Confidence drops with difficulty

Mean confidence fell steadily from clear mismatch (highest -- these decisions are easy) through strong match (very high -- all five advanced with high certainty) to borderline and edge cases (lower -- genuinely harder calls). Clear mismatch being the most confident band makes intuitive sense: it's easier to be certain someone doesn't qualify than to be certain they do.

### The most interesting individual results

**C005 (Chris Nakamura)**: the only strong match below maximum confidence. Five years is exactly at the minimum threshold and the CV notes no Kafka experience. Jev advanced him but with slightly lower confidence than the stronger candidates. That's an honest signal.

**C012 (Avery Santos)**: a Java engineer with seven years of experience and strong distributed systems background -- but no Python or Go. The lowest confidence in the borderline band. Genuinely the hardest call: experience level and systems depth are right, but the stack mismatch is real.

**C015 (Drew Hassan)**: sole backend engineer at a startup for five years -- strong ownership signals but no peer review, team collaboration or mentoring context. Jev put this in Review with high confidence, correctly identifying that human judgment is needed rather than an automatic advance or decline.

**C016 (Frankie Diaz)**: overqualified Principal Engineer with twelve years of experience applying for a Senior role. Jev didn't advance or decline -- correctly flagged this as a situation requiring human judgment. Is this a deliberate step back? A management-to-IC transition? Those questions need a conversation.

**C018 (Blair Kim)**: vague CV language with strong GitHub contributions suggesting better skills than the CV conveys. Jev flagged this for human review with very high confidence -- one of the highest-confidence Review decisions in the dataset. It correctly identified that the signals are contradictory and a human needs to look further.

**C019 (Robin Castillo)**: seven years experience but the most recent three years in PHP at a web agency. Jev declined with moderate confidence -- acknowledging the earlier Python experience but deciding the recent stack distance was too significant. Lower confidence than the clear mismatch declines, which is appropriate.

**C020 (Sage Thompson)**: career changer with no CS degree, self-taught, six years of shipped backend work. Jev put this in Review rather than declining, which is the right call -- non-traditional backgrounds deserve human assessment rather than automatic rejection.

## Latency

One Jev call per candidate, completing in well under a second each. Twenty candidates screened in well under a minute.

## The Right Architecture

Jev screening handles the easy cases automatically and surfaces the hard ones. High-confidence Advance decisions move straight to interview scheduling. High-confidence Decline decisions move straight to a rejection. Everything below the confidence threshold -- and everything in Review regardless of confidence -- goes to a human recruiter.

This is not a replacement for human judgment. It's a way to ensure that human judgment is applied where it matters most, rather than being consumed by decisions that are objectively straightforward.
