# Chapter 5: Supply Chain Disruption Response

## The Problem

A disruption hits your primary supplier. A port strike, a factory fire, an export restriction, a quality recall. You have alternative suppliers, but choosing the right one under pressure requires weighing multiple factors simultaneously: lead time, reliability, current capacity, cost and geographic risk. Get it wrong and production stalls. Get it right and nobody notices.

Rules-based fallback logic -- activate Supplier B if Supplier A is down -- works for simple scenarios. It breaks when the alternatives themselves are partially compromised, when you need to weigh trade-offs between cost and speed or when the disruption affects an entire region and your fallback suppliers share the same risk.

This chapter uses Jev to select the best available alternative supplier when a primary supplier is disrupted. One `Choice` call per disruption scenario, with the full current state of each alternative supplier in the criteria. The confidence score indicates how certain Jev is about its choice, which may not correspond to how difficult the supplier situation appears from the outside.

## The Setup

We model Vantara Devices, a fictional consumer electronics manufacturer sourcing five key components: display panels, batteries, processors, casings and cables. Each component has a primary supplier and two or three alternatives. Every supplier carries five attributes: reliability score, lead time, available capacity, cost index relative to the primary and geographic region.

Five disruption scenarios cover a range of difficulty:

- **S001 Display Panels**: port strike affecting all East Asia shipments -- two of three alternatives share the disrupted region
- **S002 Batteries**: factory fire -- clear trade-off between a faster but lower-capacity alternative and a slower but higher-capacity one
- **S003 Processors**: export restrictions on East Asia semiconductors -- one of three alternatives is also in the restricted region
- **S004 Casings**: quality recall -- two alternatives with different speed and cost profiles
- **S005 Cables**: logistics failure -- all three alternatives have a meaningful weakness, no clean winner

## One Decision per Scenario

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
        "priority":         "Minimise production delay while managing cost and risk.",
    },
    questions={
        "supplier": Choice(
            instructions=(
                "Which alternative supplier should we activate for this component "
                "given the disruption and the current state of each alternative?"
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

## What the Numbers Tell Us

### S001: Display Panels -- regional risk handled correctly

Jev chose VisionParts Europe -- the only alternative outside the disrupted East Asia region -- over two East Asian alternatives that were cheaper and faster. That's the right call: activating a supplier in the same region as the disruption simply moves the problem rather than solving it. The low confidence reflects the real cost of that decision: VisionParts is slower, less reliable and meaningfully more expensive. It's the right choice but not a comfortable one.

### S002: Batteries -- speed over capacity

Jev chose EnergyPack USA over VoltSource Japan, prioritising lead time over reliability and available capacity. The moderate confidence reflects a genuine trade-off -- the faster supplier has lower capacity and higher cost. Whether speed or capacity matters more depends on inventory levels and production schedules, context that wasn't in the state. The confidence score signals that this is a reasonable but not obvious call.

### S003: Processors -- the clearest decision

Once NexChip South Korea is excluded due to the export restrictions, SiliconWave USA is the clear choice over MicroLogic Europe: higher reliability, shorter lead time, lower cost. MicroLogic has more available capacity but loses on every other dimension. High confidence is correct -- this isn't close.

### S004: Casings -- a surprising choice

Jev chose FrameWorks Poland over ShellCraft Mexico despite Mexico being faster, cheaper and at near-full capacity. The deciding factor appears to have been reliability score, where Poland has a slight edge. The low confidence is the honest signal here: most supply chain managers would probably choose Mexico, and Jev's slight preference for Poland isn't strongly supported by the data. This is exactly the decision that should go to a human review before activation.

### S005: Cables -- speed wins when everything is compromised

With all three alternatives having meaningful weaknesses, Jev chose CableCo USA: the fastest option with the highest reliability, but limited capacity and the highest cost. It apparently weighted speed and reliability above capacity and cost, which is a defensible priority when production is stalled. Confidence was higher than expected for an "all compromised" scenario -- Jev had a clear opinion about which trade-off mattered most.

### The confidence scores didn't track our expected difficulty

Our band labels reflected our expectations: clear alternatives should produce high confidence, all-compromised scenarios should produce low confidence. The actual results inverted this -- the all-compromised cables scenario produced the highest confidence while the clear-alternative casings scenario produced the lowest.

The explanation is that our labels described the supply network structure, not how Jev experienced the decision. When Jev has a strong opinion about which factor matters most -- speed for cables, regional safety for display panels -- it acts on that opinion confidently even when the network situation looks objectively harder. When the factors are genuinely balanced -- casings, where the two alternatives are close on almost everything -- the confidence drops regardless of how the scenario was labeled.

This is a useful calibration: confidence reflects Jev's internal certainty about the decision, not the objective difficulty of the problem.

## Latency

One Jev call per scenario, completing in well under a second each. In a production supply chain system, these decisions would typically be made under time pressure but not in real time -- a disruption notification triggers an assessment within seconds rather than within milliseconds. The latency is well within any practical response window.
