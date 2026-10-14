# Conclusions

Seven domains. Seven notebooks. The same pattern every time: structured state in, typed decision out, confidence score alongside.

Looking across all seven chapters, a few things stand out.

## The confidence score is the real output

The decision -- Flag or Pass, Advance or Decline, Emergency or Self-Care -- is what gets acted on. But the confidence score is what makes the system usable.

In most chapters, the most interesting results weren't the high-confidence ones. They were the low-confidence ones: the electronics purchase that Jev passed at very low confidence, the child with a fever whose pathway moved between Urgent Care, Routine and Self-Care at the lowest confidence in the dataset, the ticket that was both a billing and a technical problem and that Jev couldn't assign to one team. These are decisions that shouldn't be automatic, and the confidence score points to them. In the later chapters, a simple threshold rule was all it took to send them to a human.

Many classifiers return a score too. What's useful here is that every decision comes with one by default. We haven't checked whether the scores are calibrated, because we had no ground truth, so we treat them as a ranking signal for where to look first.

## The middle option behaves differently across domains

Three chapters offered a middle-ground option or more than two choices at the top level: content moderation, hiring screening and medical triage. The results were different in each.

In content moderation, Flag appeared once across twenty posts. Jev leaned binary and the confidence score did the work that Flag was supposed to do -- the low-confidence Approve decisions were functionally flags, just expressed differently.

In hiring screening, Review appeared seven times. It worked as a holding zone for candidates who needed human judgment. We shouldn't credit that entirely to the domain, though: the Review option was described as an invitation to human assessment and several candidate summaries said outright that they were ambiguous.

In medical triage, all four pathways were used, with Routine used least. The instruction to prefer the more urgent pathway when in doubt is consistent with some results, but we never ran without it and several low-confidence cases went the other way, so we can't say it worked. The pathway was less important than the confidence -- a low-confidence Emergency assignment tells you something different from a high-confidence one.

The lesson isn't that three options are always better than two, or that they're always worse. It's that the middle option's utility depends on the domain and on how the options are worded. In some domains, genuine uncertainty has a natural home in the middle. In others, uncertainty shows up in the confidence score of a binary decision instead.

## Confidence depends on how you ask

The supply chain chapter showed this most clearly. In the original option order, the scenario we designed to be hardest -- all three cable suppliers compromised, no clean winner -- produced the highest confidence, and the scenarios we designed to be easier produced the lowest. When we reversed the order of the options, the cables scenario dropped to the lowest confidence and the choice itself flipped.

We can offer a reading of why: in the cables scenario, Jev may have weighted speed heavily and acted on that. But we can't separate that from presentation effects. The casings scenario shows the other side of it: Jev chose the supplier that cut against the stated priority, every time, in both orders, at low confidence.

The medical and hiring chapters saw smaller versions of the same thing. Decisions were mostly stable when we reordered the options, but confidence shifted, and for a few cases the pathway changed too. In chapter 1, adding an incident note moved confidence without changing the route.

So the confidence score doesn't measure how hard the problem is. It measures how certain Jev is about its answer in that presentation. Harder problems tend to produce lower confidence, but the two aren't the same, and the score is sensitive to details like option order and the wording of the inputs.

## The threshold is a business decision

Several chapters make the same observation in a different form. The confidence threshold that separates automatic decisions from human review isn't something Jev sets. It's something you set, based on the cost of errors in your specific context.

For fraud detection: the cost of a false positive (blocking a legitimate customer) versus a false negative (passing a fraudulent transaction) sets your threshold. For medical triage: the cost of under-triaging a serious condition versus over-triaging a minor one. For hiring: the cost of missing a strong candidate versus advancing a poor fit.

Jev gives you the infrastructure to make that trade-off explicit. The threshold is your policy, not Jev's. Because the score moves with option order, test any threshold with the options reordered before you rely on it.

## Where Jev works best

Chapter 1 starts by using Jev the wrong way. In the sequential pathfinding experiment, we asked Jev to navigate a network hop by hop. Northbound, it chose the longer Bank branch every time, with low confidence. An incident note changed its confidence but not its choice, and when we started it mid-journey it didn't backtrack. Its preference also depended on the direction of travel. That's not the shape of problem Jev is designed for.

The branch triage experiment in the same chapter shows a better shape: one decision, clear options, rich structured state. Chapter 2 extends that to a short fixed sequence of two decisions. That's the pattern we used across the book. A well-scoped question with named options and enough context to answer it -- that's where Jev earns its place.

## What we didn't test

The chapters are illustrations, not evaluations. The inputs are hand-written and sometimes carry their own cues. The samples are small. We ran most notebooks once and the harder cases a handful of times, and we never compared a decision against a correct answer. A real deployment would need all of that: realistic data, expected answers, repeat runs, checks across option orders and, in hiring and medicine, bias auditing, clinical or legal review and human oversight.

The seven chapters in this book are a range of examples. The underlying pattern is always the same.
