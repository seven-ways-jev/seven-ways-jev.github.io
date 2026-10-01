# Chapter 1: Transit Routing

## The Northern Line Branch

The London Underground's Northern line presents a genuine decision problem. Between Kennington and Euston the line splits into two branches that don't share a single station until they both reach Euston:

- **Via Charing Cross** (8 stops): a slightly shorter route running through the West End
- **Via Bank** (9 stops): a slightly longer route running through the City

On a quiet day with both branches running normally, the right choice is obvious: take Charing Cross, it's one stop shorter. But the moment something goes wrong on the network, the right choice depends on live information that a static algorithm doesn't have.

This chapter uses that branch decision as a vehicle for exploring two questions. First: can Jev navigate a network hop by hop, making a fresh decision at every step? Second: is that even the right way to use it?

What we find along the way tells us something useful about where Jev actually earns its place.

## The Data

We use the real London Underground dataset: around 350 stations, all lines, with geographic coordinates for each station. We load it into a NetworkX `MultiGraph` directly from CSV files. A `MultiGraph` is essential here because many stations are served by multiple lines and a plain `Graph` would overwrite one line's edges with another's when the same station pair appears more than once.

Each station is a node carrying its coordinates and zone. Each connection is an edge carrying the line name. The dataset includes all its real-world naming quirks: "Kings Cross St. Pancras" has no apostrophe, and "Elephant and Castle" is spelled out in full.

The notebook verifies all branch station names against the graph before running anything.

## Part 1: Jev as a Sequential Planner

Our first approach was to use Jev as a navigation agent: query the graph for the current station's neighbors, pass the live state to Jev, let it pick the next hop, and repeat until we reach Euston.

This is an appealing idea. It means the route adapts at every step rather than being fixed at the start. If something changes mid-journey, the very next call to Jev incorporates it.

### Baselines

Before introducing Jev, we establish two baselines using NetworkX's `shortest_path()`.

The unconstrained version finds the globally optimal route across all lines: **6 hops** via Waterloo, Westminster, Green Park, Oxford Circus and Warren Street. That crosses the Jubilee and Victoria lines -- a real Northern line passenger wouldn't consider this a valid route.

Adding a Northern-line-only constraint finds the correct result: **8 hops** via the Charing Cross branch (Waterloo, Embankment, Charing Cross, Leicester Square, Tottenham Court Road, Goodge Street, Warren Street). Both queries run in well under a millisecond.

The key point: `shortest_path()` is exact and context-naive by design. It finds the best answer within whatever constraints you give it. The constraints are your job.

### What happened

Across multiple northbound runs from Kennington to Euston, Jev consistently chose the **Bank branch** -- 9 hops every time, even though the Charing Cross branch is one stop shorter. Hop-1 confidence was consistently low throughout.

We then injected a disruption. At Leicester Square, four stops into the Charing Cross route, the incident note reported Warren Street suspended ahead. The hypothesis was that Jev might decide to push on toward the disruption or backtrack and reroute via Bank. Instead, it made no change at all. Same Bank branch, same 9 hops, same confidence range -- because it had already committed to Bank at Kennington before Leicester Square was ever in the path.

The reverse journey (Euston to Kennington, southbound) was more consistent but in the opposite direction: Charing Cross branch every time, with noticeably higher confidence. The same network, the same graph, a different direction and a different stable preference. That directional asymmetry is almost certainly a training data artifact, not geographic reasoning.

The most revealing experiment was starting from Waterloo, which sits on the Charing Cross branch with no fork to navigate. Without the anchor of a clear branch decision, hop counts varied widely across 5 runs and confidence dropped significantly. The structured problem that had produced consistent results at Kennington became underdetermined when the framing changed.

### What this tells us

TypeSafe are explicit about what Jev is designed for: fast, atomic, well-scoped judgments -- "the kind a highly knowledgeable person could make in a few seconds given the right context." A nine-hop sequential plan isn't that. It's a planning problem, and `shortest_path()` solves planning problems in under a millisecond.

None of this is a criticism of Jev. It's a mismatch between problem shape and tool design.

## Part 2: Branch Triage

The fairer test is to ask Jev for one decision: at Kennington, which branch should a passenger take?

We store disruption state directly on the graph nodes, query both branches' current state, build a plain-language summary of each and pass both to Jev as a single `Choice` question with two options. One call, clear state, two options. That's the shape of problem Jev was built for.

We run this across four disruption severities, three times each:

| Scenario | Choice | Unanimous |
|---|---|---|
| No disruption | Charing Cross | Yes |
| Mild: Goodge Street delayed 5 min | Bank | Yes |
| Severe: Warren Street suspended | Bank | Yes |
| Critical: Warren Street and Goodge Street out | Bank | Yes |

This is exactly the behavior TypeSafe describe. With both branches clear, Jev chose Charing Cross -- the shorter route -- with solid confidence. A 5-minute delay on a station that can't be bypassed was enough to flip the decision to Bank and confidence rose because the signal got clearer. A full suspension produced even higher confidence and two suspended stations produced the highest confidence of all. Every run was unanimous. Latency was well under 500 milliseconds per call -- a fraction of the time the sequential pathfinder took per complete journey.

The confidence with no disruption is worth noting. Jev isn't overconfident. Both branches are valid, Charing Cross is one stop shorter and the moderate confidence reflects a real preference -- but it also signals that Bank isn't a bad choice, which is true.

## The Right Architecture

A shortest-path algorithm is exact, instant and optimal within whatever constraints you give it. Add a line constraint and it finds the Northern-line-only optimal in the same single call. It has no concept of disruption, live state, or calibrated uncertainty. Those aren't gaps -- they're design choices.

Jev is the right tool for the judgment calls that sit on top of that: which of these valid options is better right now, given what's happening? That question requires weighing uncertain information and returning a decision with calibrated confidence. That's what Jev is built for.

The algorithm finds the route. Jev decides which route is worth taking.
