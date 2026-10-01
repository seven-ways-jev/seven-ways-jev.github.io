# Chapter 7: Medical Symptom Triage

## ⚠ Critical Disclaimer

**This chapter is for illustrative and educational purposes only.**

The presentations, decisions and confidence scores in this chapter are fictional. Jev is not a medical device, has not been clinically validated and must not be used to make or influence real healthcare decisions.

**If you or someone else is experiencing a medical emergency, call the emergency services immediately.**

Any real application of AI in clinical triage requires regulatory approval, clinical validation, human oversight and clear accountability for every decision. Nothing in this chapter should be interpreted as guidance for real-world medical use.

---

## The Problem

A patient contacts a healthcare service. They describe their symptoms. Someone -- or something -- has to decide how urgently they need to be seen.

Traditional triage uses structured protocols: numerical scores, decision trees, trained clinicians working through standardized questions. These work well but require skilled human time. At scale -- a busy out-of-hours service, a telehealth platform handling thousands of daily contacts -- the bottleneck is always clinical capacity.

This chapter uses Jev to assign each patient presentation to one of four care pathways: Emergency, Urgent Care, Routine or Self-Care. One `Choice` call per presentation, patient symptoms and context in the state, calibrated confidence alongside the decision.

The goal is not to replace clinical triage. It is to illustrate where confidence-scored structured decisions can help: routing clear cases quickly and flagging uncertain ones for human review.

## The Setup

We generate 20 fictional patient presentations across four bands. Each carries the reported symptoms, patient age, sex, relevant medical history and symptom duration.

The four pathways reflect standard triage levels:

- **Emergency**: potentially life-threatening, call emergency services immediately
- **Urgent Care**: needs clinical assessment within hours
- **Routine**: warrants clinical review but not urgent, GP appointment within days
- **Self-Care**: minor, can be managed at home

The triage instructions include a conservative bias: when in doubt, assign the more urgent pathway. In medical triage, the cost of under-triaging a serious condition is far higher than the cost of over-triaging a minor one.

## One Decision per Presentation

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
| P005 | 45 | Clear emergency | Urgent Care |
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
| P017 | 7 | Edge | Self-Care |
| P018 | 31 | Edge | Urgent Care |
| P019 | 25 | Edge | Emergency |
| P020 | 55 | Edge | Emergency |

## What the Numbers Tell Us

### Clean at the extremes

All five clear self-care presentations were correctly assigned to Self-Care with high confidence. Four of the five clear emergencies were correctly assigned to Emergency at maximum confidence. The fifth is the most interesting result of the chapter.

### P005: haemoptysis -- the one debatable clear emergency

P005, a 45-year-old male smoker who has coughed up blood for the first time after a three-week cough, was assigned to Urgent Care rather than Emergency -- with very low confidence. Coughing up blood in a long-term smoker is a serious red flag warranting urgent investigation, but it isn't immediately life-threatening in the way a heart attack or stroke is. The low confidence reflects genuine clinical uncertainty about whether this requires an ambulance right now or an urgent same-day appointment. Arguable either way and the low confidence correctly signals this for human review.

### Conservative bias working as intended

The instruction to "assign the more urgent pathway when in doubt" is visible in the edge case results. Three edge case presentations were assigned Emergency:

**P016** (82-year-old on blood thinners who has fallen): even without a head injury, the blood thinner history and age significantly elevate the risk of internal bleeding from what might otherwise look like a minor fall.

**P019** (25-year-old with panic attack history, symptoms overlap with cardiac and respiratory emergency): the prior diagnosis points toward a panic attack, but the symptoms are indistinguishable from a cardiac event without clinical assessment. Jev assigned Emergency with moderate confidence -- defensible given the instruction.

**P020** (55-year-old hypertensive with new blurry vision): sudden visual changes in a hypertensive patient could indicate a hypertensive emergency or a transient ischaemic attack. The conservative bias pushed this to Emergency.

### P017: the most concerning result

The 7-year-old with a 39.2C fever for 24 hours was assigned Self-Care -- with the lowest confidence of any result in the dataset. In most paediatric triage protocols, a persistent fever at that temperature in a child would warrant at minimum a Routine appointment and more commonly Urgent Care. The very low confidence is the correct signal: this decision should not stand without clinical review.

### Confidence tracks uncertainty correctly

Mean confidence by band drops steadily from clear self-care (highest) through clear emergency to ambiguous and edge cases (lowest). Emergency and Self-Care are the high-confidence pathways -- Jev is more certain at both ends of the urgency spectrum than in the middle. Urgent Care and Routine sit at lower mean confidence, reflecting that these are the genuinely harder calls.

## Latency

One Jev call per presentation, completing in well under a second each. Twenty presentations triaged in well under a minute.

## The Right Architecture

In a real system, every decision would be reviewed by a qualified clinician before any action is taken. The confidence score determines the priority of that review: high-confidence decisions can be processed quickly, low-confidence decisions need immediate human attention.

The value Jev adds is not replacing clinical judgment -- it is making the uncertain cases visible. A call handler working through a hundred contacts can't give equal attention to all of them. A confidence score flags the presentations where attention matters most.

**Reminder: this chapter is for illustrative purposes only. Never use Jev or any AI system as a substitute for clinical judgment.**
