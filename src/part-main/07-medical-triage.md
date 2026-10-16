# Chapter 7: Medical symptom triage

## ⚠ Critical disclaimer

**This chapter is for illustrative and educational purposes only.**

The presentations, decisions and confidence scores in this chapter are fictional. Jev is not a medical device, has not been clinically validated and must not be used to make or influence real healthcare decisions.

**If you or someone else is experiencing a medical emergency, call the emergency services immediately.**

Any real application of AI in clinical triage requires regulatory approval, clinical validation, human oversight and clear accountability for every decision. Nothing in this chapter should be interpreted as guidance for real-world medical use.

---

## The problem

A patient contacts a healthcare service. They describe their symptoms. Someone -- or something -- has to decide how urgently they need to be seen.

Traditional triage uses structured protocols: numerical scores, decision trees, trained clinicians working through standardized questions. These work well but require skilled human time. At scale -- a busy out-of-hours service, a telehealth platform handling thousands of daily contacts -- the bottleneck is always clinical capacity.

This chapter uses Jev to assign each patient presentation to one of four care pathways: Emergency, Urgent Care, Routine or Self-Care. One `Choice` call per presentation, patient symptoms and context in the state, a confidence score alongside the decision.

The goal is not to replace clinical triage. It is to illustrate where confidence-scored structured decisions can help: routing clear cases quickly and flagging uncertain ones for human review.

## The setup

We write 20 fictional patient presentations across four bands:

- **Clear emergency** (4 presentations): textbook emergencies such as a suspected heart attack, stroke or anaphylaxis
- **Clear self-care** (5 presentations): minor, self-limiting problems
- **Ambiguous** (6 presentations): reasonable people could triage them differently
- **Edge cases** (5 presentations): unusual profiles such as an elderly patient on blood thinners who has fallen, a child with a fever or a panic attack that looks like a cardiac event

Each presentation carries the reported symptoms, patient age, sex, relevant medical history and symptom duration. The band label is ours and isn't passed to Jev. We first placed P005, who is coughing up blood, among the clear emergencies, then moved it to the ambiguous band. It's a serious red flag but not clearly an emergency, and the first run's result showed us the label was wrong.

The four pathways reflect standard triage levels:

- **Emergency**: potentially life-threatening, call emergency services immediately
- **Urgent Care**: needs clinical assessment within hours
- **Routine**: warrants clinical review but not urgent, GP appointment within days
- **Self-Care**: minor, can be managed at home

The triage instructions include a conservative bias: when in doubt, assign the more urgent pathway. In medical triage, the cost of under-triaging a serious condition is far higher than the cost of over-triaging a minor one.

## One decision per presentation

One `Choice` call per presentation. All available patient context goes into the state.

```python
response = client.system_one(
    state={
        "patient_age":       presentation["age"],
        "patient_sex":       presentation["sex"],
        "medical_history":   presentation["medical_history"],
        "reported_symptoms": presentation["symptoms"],
        "symptom_duration":  presentation["duration"],
    },
    questions={
        "pathway": Choice(
            instructions=(
                "Based on the reported symptoms, patient age, medical history "
                "and symptom duration, which care pathway is most appropriate? "
                "When in doubt, assign the more urgent pathway."
            ),
            criteria=CARE_PATHWAYS,
        )
    },
)
```

## Results

| ID | Age | Band | Pathway |
|---|---|---|---|
| P001 | 58 | Clear emergency | Emergency |
| P002 | 67 | Clear emergency | Emergency |
| P003 | 34 | Clear emergency | Emergency |
| P004 | 22 | Clear emergency | Emergency |
| P005 | 45 | Ambiguous | Urgent Care |
| P006 | 28 | Clear self-care | Self-Care |
| P007 | 35 | Clear self-care | Self-Care |
| P008 | 19 | Clear self-care | Self-Care |
| P009 | 42 | Clear self-care | Self-Care |
| P010 | 31 | Clear self-care | Self-Care |
| P011 | 44 | Ambiguous | Routine |
| P012 | 52 | Ambiguous | Urgent Care |
| P013 | 38 | Ambiguous | Urgent Care |
| P014 | 60 | Ambiguous | Routine |
| P015 | 29 | Ambiguous | Urgent Care |
| P016 | 82 | Edge | Emergency |
| P017 | 7 | Edge | Urgent Care |
| P018 | 31 | Edge | Urgent Care |
| P019 | 25 | Edge | Emergency |
| P020 | 55 | Edge | Emergency |

## What the numbers tell us

The confidence scores change from run to run, so we describe patterns rather than exact values.

### Clean at the extremes

All four clear emergencies went to Emergency at essentially maximum confidence. All five clear self-care presentations went to Self-Care with very high confidence. These cases are also the ones we wrote with the strongest cues ("crushing chest pain", "worst of her life", "mild", "rated 3/10"), so their high confidence partly reflects how easy we made them. Real contacts are rarely this tidy.

### P005: Coughing up blood

P005, a 45-year-old male smoker who has coughed up blood for the first time after a three-week cough, went to Urgent Care rather than Emergency, with low confidence. In our reading, this is a serious red flag warranting prompt investigation but not obviously an ambulance case. Jev doesn't explain its choice, so that reading is ours. Jev chose Urgent Care every time we repeated the call, in both option orders. A stable, low-confidence decision is exactly the kind a human should look at.

### The conservative bias is not demonstrated

The instruction says to assign the more urgent pathway when in doubt. The results are mixed. Three edge cases went to Emergency (P016, P019 and P020), which is consistent with the bias. But P005, P011 and P014 went to a less urgent pathway while sitting at low confidence, and P017 (below) drifted between three pathways. We also never ran the triage without the instruction, so we can't say how much of any result comes from it. We've also only got our own reading of why Jev chose what it chose: the age and blood thinners for P016, the overlap between panic and cardiac symptoms for P019, and the possible blood pressure link to the blurry vision for P020. Treat those as plausible explanations, not findings.

### P017: The least stable result

The 7-year-old with a 39.2C fever for 24 hours had the lowest confidence of any presentation in every run. Its pathway wasn't stable either. In the first run it went to Urgent Care. Across the repeats, the original option order gave Urgent Care in most runs and Self-Care in one, and the reversed order gave Routine in most runs. The input also reads reassuringly (alert, eating and drinking normally, no rash), so this is a hard case. We'd argue that, given the instruction to err toward urgency, a decision that can land on Self-Care at all shouldn't stand without clinical review. The low confidence is the signal that it needs one.

### Option order matters

Chapters 5 and 6 tested the order of the options, and we did the same here. The pathways are normally listed from most to least urgent. We reversed that and ran each ambiguous and edge presentation five times in each order.

Within a single order, the pathway was almost always stable. Between orders, it wasn't always:

- **P017** moved from Urgent Care to Routine, as described above.
- **P020** was Emergency in every run in the original order, but went to Urgent Care in most runs when reversed. The Emergency result for the blurry-vision patient depends on how we listed the options.
- **P019** stayed at Emergency, but its confidence dropped noticeably when the order was reversed.
- **P005** stayed at Urgent Care, but its confidence rose when reversed.
- Several others (P011, P012, P014, P018) kept their pathway but shifted somewhat in confidence.

Confidence is sensitive to presentation, not just to the patient. That matters most near a fixed threshold. Reversed, P014's confidence sat above 0.70, which is the threshold we use below, while in the original order it sat below.

### Confidence tracks the clarity of the input

Mean confidence was highest for the two clear bands, lower for the ambiguous band and lowest for the edge cases. By pathway, Emergency and Self-Care had the higher means and Urgent Care and Routine the lower ones, but Routine has only two presentations, so we wouldn't read much into it. That pattern is partly built in, because we wrote the clear cases to be clear. What we can say is that the cases we thought were hard were the ones Jev was least sure about.

### Routing by confidence

We route anything below a confidence of 0.70 to a human, whichever pathway Jev chose. In our run, 10 of 20 presentations went to a human: five of the six ambiguous presentations (all but P013) and all five edge cases. The other 10 were the nine clear presentations and P013, which went to Urgent Care. P005 and P017 were both in the human group.

The risk to watch is the opposite case: a presentation that is under-triaged with high confidence would pass straight through. We saw none here, but we have no ground truth, and the high-confidence cases were the ones we made easy.

## Latency

One Jev call per presentation, completing in well under a second each. Twenty presentations triaged in a few seconds.

## The right architecture

In a real system, every decision would be reviewed by a qualified clinician before any action is taken. The confidence score determines the priority of that review: low-confidence decisions need prompt human attention, whichever pathway Jev chose, and high-confidence decisions can be processed quickly.

The value Jev adds is not replacing clinical judgment -- it is making the uncertain cases visible. A call handler working through a hundred contacts can't give equal attention to all of them. A confidence score flags the presentations where attention matters most. As the option-order results show, the score depends on how the question is asked, so any threshold should be tested that way before anyone relies on it.

**Reminder: this chapter is for illustrative purposes only. Never use Jev or any AI system as a substitute for clinical judgment.**
