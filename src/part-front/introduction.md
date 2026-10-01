# Introduction

Most AI models are designed to converse. You give them a prompt, they give you text back. That text might be excellent -- detailed, nuanced, well-written -- but it's freeform. If you want to act on it programmatically, you have to parse it, validate it and hope the format holds across different inputs.

[Jev](https://typesafe.ai), from TypeSafe AI, is built on a different premise. It's a structured decision model: you give it a state object and a set of typed questions and it returns typed answers with calibrated confidence scores. No parsing required. No format negotiation. Just a decision and a number that tells you how certain that decision is.

TypeSafe describe Jev as a "System One" model -- built for the kind of fast, atomic, well-scoped judgments that a highly knowledgeable person could make in a few seconds given the right context. Not a general-purpose reasoner. Not a planner. A decision maker.

The core primitive is the `Choice`: a question with named options, each described in plain language, plus an instruction telling Jev what to weigh. Each `Choice` returns the selected option and a confidence score between 0 and 1. That confidence score is what makes Jev useful in production -- not just as a curiosity, but as an operational signal. A high-confidence decision routes automatically. A low-confidence decision goes to a human reviewer. The threshold between them is a business decision, not a technical one and it's tunable.

## What this book covers

Seven domains, seven notebooks, one pattern: structured state in, typed decision out, confidence score alongside.

The chapters are independent. You can read them in any order and run any notebook without having read the others. Each is self-contained, with synthetic data usually generated inside the notebook and no external dependencies beyond a TypeSafe API key.

The domains span a deliberate range of difficulty and context:

- **Transit routing**: a graph problem where we first try to use Jev the wrong way, then the right way
- **Customer support triage**: two sequential decisions per ticket, team then priority
- **Financial fraud detection**: a binary flag-or-pass decision with a three-tier response based on confidence
- **Content moderation**: three options and a finding about how Jev uses (or doesn't use) the middle one
- **Supply chain disruption**: selecting an alternative supplier under pressure, with a surprising confidence pattern
- **Hiring screening**: three options again, with a much more even distribution than content moderation
- **Medical symptom triage**: four pathways, a conservative bias built into the instructions and the strongest disclaimer in the book

## What you'll see across chapters

A few patterns emerge across all seven chapters and are worth keeping in mind from the start.

**The confidence score matters more than the decision for uncertain cases.** Across every chapter, the low-confidence decisions are the interesting ones -- not because Jev got them wrong, but because they're the decisions where no system should be acting automatically. High confidence means act. Low confidence means review. The decision itself is secondary.

**Three options work better in some domains than others.** Hiring produced a genuine spread across Advance, Review and Decline. Content moderation produced almost no Flag decisions -- Jev leaned binary and the confidence score did the work that Flag was supposed to do.

**Confidence reflects internal certainty, not objective difficulty.** The supply chain chapter shows this most clearly: the scenario we expected to be hardest produced the highest confidence, because Jev had a strong opinion about which trade-off mattered. The one we expected to be easy produced the lowest confidence, because the options were genuinely close.

**The threshold is always yours to set.** Every chapter ends with some version of the same observation: the confidence threshold that separates automatic decisions from human review is a business decision. Jev provides the infrastructure; you set the policy.

## Running the notebooks

Each notebook installs its own dependencies, generates its own data and runs end to end with a single environment variable:

```shell
export TYPESAFE_API_KEY="your-api-key"

jupyter notebook 01_northern_line.ipynb
```

Chapter 1 additionally uses CARTO map tiles for the Folium maps. A CARTO API key is optional -- the notebook falls back to OpenStreetMap tiles if none is provided:

```shell
export CARTO_API_KEY="your-api-key"
```

Chapter 1 also requires three CSV files in a `data/` directory alongside the notebook. All other chapters generate their data internally.
