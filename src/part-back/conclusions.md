# Conclusion

Seven domains. Seven notebooks. The same pattern every time: structured state in, typed decision out, confidence score alongside.

Looking across all seven chapters, a few things stand out.

## The confidence score is the real output

The decision -- Flag or Pass, Advance or Decline, Emergency or Self-Care -- is what gets acted on. But the confidence score is what makes the system trustworthy.

In every chapter, the most interesting results weren't the high-confidence ones. They were the low-confidence ones: the electronics purchase that Jev passed at near-zero confidence, the child with a fever assigned Self-Care at the lowest confidence in the dataset, the billing-or-technical ticket that Jev couldn't decide who owned. These are the decisions that should never be automatic and the confidence score identifies them without any additional logic.

A binary classifier gives you a decision. Jev gives you a decision and a signal about how much to trust it. That second thing is what makes the difference between a system that routes automatically and one that routes intelligently.

## The middle option behaves differently across domains

Three chapters used three-option choices. The results were different in each.

In content moderation, Flag appeared once across twenty posts. Jev leaned binary and the confidence score did the work that Flag was supposed to do -- the low-confidence Approve decisions were functionally flags, just expressed differently.

In hiring screening, Review appeared seven times. The middle option worked as intended: a genuine holding zone for candidates who needed human judgment rather than an automatic decision.

In medical triage, four pathways produced a spread across all of them, but with the conservative bias pushing toward the more urgent option when Jev was uncertain. The pathway was less important than the confidence -- a low-confidence Emergency assignment tells you something different from a high-confidence one.

The lesson isn't that three options are always better than two, or that they're always worse. It's that the middle option's utility depends on the domain. In some domains, genuine uncertainty has a natural home in the middle. In others, uncertainty shows up in the confidence score of a binary decision instead.

## Confidence reflects internal certainty, not objective difficulty

The supply chain chapter showed this most clearly. The scenario designed to be hardest -- all three cable suppliers compromised, no clean winner -- produced the highest confidence. The scenario designed to be easiest -- two casings suppliers with clear profiles -- produced the lowest.

The explanation is that Jev had a strong opinion about which factor mattered most in the cables scenario (speed) and acted on that opinion confidently. In the casings scenario, the two alternatives were close on almost everything and Jev genuinely couldn't separate them.

This pattern appeared in every chapter. The confidence score doesn't measure how hard the problem is. It measures how certain Jev is about its answer. Those two things correlate -- harder problems tend to produce lower confidence -- but they're not the same and the medical triage chapter shows what happens when they diverge: Jev assigned a child's fever to Self-Care at very low confidence, correctly signalling that a human needed to review that decision even though the symptoms were objectively mild.

## The threshold is always a business decision

Every chapter ends with the same observation in a different form. The confidence threshold that separates automatic decisions from human review isn't something Jev sets. It's something you set, based on the cost of errors in your specific context.

For fraud detection: the cost of a false positive (blocking a legitimate customer) versus a false negative (passing a fraudulent transaction) sets your threshold. For medical triage: the cost of under-triaging a serious condition versus over-triaging a minor one. For hiring: the cost of missing a strong candidate versus advancing a poor fit.

Jev gives you the infrastructure to make that trade-off explicit. The threshold is your policy, not Jev's.

## Where Jev works best

Chapter 1 is the one chapter that starts by using Jev the wrong way. The sequential pathfinding experiment -- asking Jev to navigate a network hop by hop, making a dependent chain of decisions -- produced inconsistent routes and failed to respond to live disruption information. Not because Jev is a weak model, but because that's not the shape of problem it was built for.

The branch triage experiment in the same chapter shows the right shape: one decision, clear options, rich structured state. That's the pattern that worked across all seven domains. A well-scoped question with named options and enough context to answer it -- that's where Jev earns its place.

The seven chapters in this book are a range of examples. The underlying pattern is always the same.
