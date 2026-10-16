# Chapter 5: Supply chain disruption response

## The problem

A disruption hits your primary supplier. A port strike, a factory fire, an export restriction, a quality recall. You have alternative suppliers, but choosing the right one under pressure requires weighing multiple factors simultaneously: lead time, reliability, current capacity, cost and geographic risk. Get it wrong and production stalls. Get it right and nobody notices.

Rules-based fallback logic -- activate Supplier B if Supplier A is down -- works for simple scenarios. It breaks when the alternatives themselves are partially compromised, when you need to weigh trade-offs between cost and speed or when the disruption affects an entire region and your fallback suppliers share the same risk.

This chapter uses Jev to select the best available alternative supplier when a primary supplier is disrupted. One `Choice` call per disruption scenario, with the full current state of each alternative supplier in the criteria. The confidence score indicates how certain Jev is about its choice, which may not correspond to how difficult the supplier situation appears from the outside.

## The setup

We model Vantara Devices, a fictional consumer electronics manufacturer sourcing five key components: display panels, batteries, processors, casings and cables. Each component has a primary supplier and two or three alternatives. Every supplier carries five attributes: reliability score, lead time, available capacity, cost index relative to the primary and geographic region.

Five disruption scenarios cover a range of difficulty:

- **S001 Display Panels**: port strike affecting all East Asia shipments -- two of three alternatives share the disrupted region
- **S002 Batteries**: factory fire -- clear trade-off between a faster but lower-capacity alternative and a slower but higher-capacity one
- **S003 Processors**: export restrictions on East Asia semiconductors -- one of three alternatives is also in the restricted region
- **S004 Casings**: quality recall -- one faster, cheaper alternative with near-full capacity and one slightly more reliable alternative
- **S005 Cables**: logistics failure -- all three alternatives have a meaningful weakness, no clean winner

## One decision per scenario

Each scenario produces one `Choice` call. The disruption goes into the state; each alternative supplier becomes a criterion described in plain language.

```python
criteria = {
    name: describe_supplier(name, attrs)
    for name, attrs in alternatives.items()
}

response = client.system_one(
    state={
        "component":        component,
        "primary_supplier": primary["name"],
        "disruption":       disruption,
        "priority":         "Minimize production delay while managing cost and risk.",
    },
    questions={
        "supplier": Choice(
            instructions=(
                "Which alternative supplier should we activate for this component "
                "given the disruption and the current state of each alternative? "
                "Consider lead time, reliability, available capacity, cost and "
                "any shared regional risk with the disrupted primary supplier."
            ),
            criteria=criteria,
        )
    },
)
```

## Results

| Scenario | Component | Chosen |
|---|---|---|
| S001 | Display Panels | VisionParts Europe |
| S002 | Batteries | EnergyPack USA |
| S003 | Processors | SiliconWave USA |
| S004 | Casings | FrameWorks Poland |
| S005 | Cables | CableCo USA |

Each scenario was run once for this table. We then repeated every scenario five times with the options listed in the original order and five times reversed, which we come back to below. Confidence scores vary between runs, so we describe the patterns and leave the exact values to the notebook.

## What the numbers tell us

### S001: Display panels -- routing around a shared region

Jev chose VisionParts Europe -- the only alternative outside the disrupted East Asia region -- over two East Asian alternatives that were cheaper and faster. That's a sensible call: activating a supplier in the same region as the disruption simply moves the problem rather than solving it. The scenario text and the instructions both point Jev at regional risk, so this is as much careful reading as discovery.

Confidence was low. Part of that is the trade-off: VisionParts is slower, less reliable and meaningfully more expensive. But the repeat runs below show the number also depends on the order the options are listed in, so we don't read too much into it.

### S002: Batteries -- speed over capacity

Jev chose EnergyPack USA over VoltSource Japan, prioritizing lead time over reliability and available capacity. The moderate confidence reflects a genuine trade-off -- the faster supplier has lower capacity and higher cost. Whether speed or capacity matters more depends on inventory levels and production schedules, context that wasn't in the state. The confidence score signals that this is a reasonable but not obvious call.

### S003: Processors -- the clearest decision

Once NexChip South Korea is excluded due to the export restrictions, SiliconWave USA is the clear choice over MicroLogic Europe: higher reliability, shorter lead time, lower cost. MicroLogic has more available capacity but loses on every other dimension. High confidence is correct -- this isn't close.

### S004: Casings -- a surprising choice

Jev chose FrameWorks Poland over ShellCraft Mexico, even though Mexico is faster, cheaper and at near-full capacity. Poland's only edge is a slightly higher reliability score. The scenario's stated priority is to minimize production delay, which points to Mexico, so this is the one choice that cuts against the brief. Confidence was low, and Jev chose Poland every time, in both option orders. This is exactly the decision that should go to human review before activation.

### S005: Cables -- speed wins, unless the order changes

With all three alternatives having meaningful weaknesses, Jev chose CableCo USA: the fastest option, with the second-highest reliability, but limited capacity and the highest cost. It weighted speed above capacity and cost, which is a defensible priority when production is stalled. In the original order confidence was higher than expected for an "all compromised" scenario.

S005 is also the one scenario where the choice itself depended on the order of the options. With the options reversed, Jev usually chose FlexLink Germany, the most reliable option, and with much lower confidence.

### Option order matters

We repeated every scenario five times with the options in the original order and five times reversed. Within an order the choices were stable. Across orders the choice was identical in four of the five scenarios, but the confidence moved: S001 became much more confident with the options reversed, while S004 and S005 became less so, and S005's choice flipped.

The practical lesson is that the confidence number is sensitive to how the options are presented, so it shouldn't be compared across prompts that list the options differently. If a choice or its confidence changes when you reorder the options, treat that as a reason to send the decision to a human.

### The confidence scores didn't track our expected difficulty

Our band labels reflected our expectations: clear alternatives should produce high confidence, all-compromised scenarios should produce low confidence. In the original order the results inverted this -- the all-compromised cables scenario averaged the highest confidence and the clear-alternative scenarios (batteries and casings) the lowest, while the two regional risk scenarios sat at opposite ends, display panels low and processors very high.

With the options reversed, the picture changed: cables dropped to the lowest confidence and display panels rose. So the apparent inversion is partly an artifact of option order, and with five scenarios we can't say much more. One reading is that our labels described the network structure rather than how Jev experienced the decision, but we can't separate that from presentation effects.

What we can say is that confidence is a signal about the decision as presented, not an objective measure of how hard the problem is.

## Latency

One Jev call per scenario, completing in well under a second each. In a production supply chain system, these decisions would typically be made under time pressure but not in real time -- a disruption notification triggers an assessment within seconds rather than within milliseconds. The latency is well within any practical response window.

One caution: we didn't check any choice against an expected answer, so the scores show where Jev is unsure, not how often it's right.
