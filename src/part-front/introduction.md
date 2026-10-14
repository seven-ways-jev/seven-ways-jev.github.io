# Introduction

Most AI models are designed to converse. You give them a prompt, they give you text back. That text might be excellent -- detailed, nuanced, well-written -- but it's freeform. If you want to act on it programmatically, you have to parse it, validate it and hope the format holds across different inputs.

[Jev](https://typesafe.ai), from TypeSafe AI, is built on a different premise. It's a structured decision model: you give it a state object and a set of typed questions and it returns typed answers with confidence scores. No parsing required. No format negotiation. Just a decision and a number that tells you how certain that decision is.

TypeSafe describe Jev as a "System One" model -- built for the kind of fast, atomic, well-scoped judgments that a highly knowledgeable person could make in a few seconds given the right context. Not a general-purpose reasoner. Not a planner. A decision maker.

The core primitive is the `Choice`: a question with named options, each described in plain language, plus an instruction telling Jev what to weigh. Each `Choice` returns the selected option and a confidence score between 0 and 1. That confidence score is what makes Jev useful in production -- not just as a curiosity, but as an operational signal. A high-confidence decision is a candidate for automatic handling. A low-confidence decision goes to a human reviewer. The threshold between them is a business decision, not a technical one and it's tunable.

## What this book covers

Seven domains, seven notebooks, one pattern: structured state in, typed decision out, confidence score alongside.

The chapters are mostly independent. You can read them in any order and run any notebook without having read the others. Each is self-contained, with the data written by hand inside the notebook and no external dependencies beyond a TypeSafe API key.

The domains span a deliberate range of difficulty and context:

- **Transit routing**: a graph problem where we first try to use Jev the wrong way, then the right way
- **Customer support triage**: two sequential decisions per ticket, team then priority
- **Financial fraud detection**: a binary flag-or-pass decision with a three-tier response based on confidence
- **Content moderation**: three options and a finding about how Jev uses (or doesn't use) the middle one
- **Supply chain disruption**: selecting an alternative supplier under pressure, with a confidence pattern that depends on the order of the options
- **Hiring screening**: three options again, with a much more even distribution than content moderation
- **Medical symptom triage**: four pathways, a conservative bias built into the instructions and the strongest disclaimer in the book

## What you'll see across chapters

A few patterns emerge across all seven chapters and are worth keeping in mind from the start.

**The confidence score matters for uncertain cases.** In most chapters, the low-confidence decisions are the interesting ones -- not because Jev got them wrong, but because they're the decisions where no system should be acting automatically. High confidence is a candidate for automatic handling. Low confidence means review.

**Three options behave differently in different domains.** Hiring produced a spread across Advance, Review and Decline, although part of that comes from how we worded the options and wrote the candidates. Content moderation produced almost no Flag decisions -- Jev leaned binary and the confidence score did the work that Flag was supposed to do.

**Confidence isn't the same as difficulty, and it depends on how you ask.** In the supply chain chapter, the scenario we expected to be hardest produced the highest confidence when the options were listed in one order and the lowest when we reversed them. Similar order effects showed up in the hiring and medical chapters. Treat the score as a signal that's sensitive to how the question is presented.

**The threshold is yours to set.** Several chapters make the same observation: the confidence threshold that separates automatic decisions from human review is a business decision. Jev provides the infrastructure; you set the policy.

## How to read the results

A few limits apply to every chapter:

- The inputs are hand-written, and we sometimes put cues in them, so some of the high-confidence results are easier than real data would be.
- We ran each notebook once, with repeat runs for the harder cases in the later chapters. Confidence scores change from run to run, so the chapters describe patterns rather than exact values.
- We didn't check any decision against an expected answer, so we can't say whether the scores are calibrated.
- The order of the options can change the confidence and sometimes the decision.
- The medical and hiring chapters are illustrations, not guidance for real use.

## Running the notebooks

Each notebook installs its own dependencies, uses data defined in the notebook and runs end to end with a single environment variable:

```shell
export TYPESAFE_API_KEY="your-api-key"

jupyter notebook 01_northern_line.ipynb
```

Chapter 1 additionally uses CARTO map tiles for the Folium maps. A CARTO API key is optional -- the notebook falls back to OpenStreetMap tiles if none is provided:

```shell
export CARTO_API_KEY="your-api-key"
```

Chapter 1 also requires three CSV files in a `data/` directory alongside the notebook. All other chapters define their data inside the notebook.
