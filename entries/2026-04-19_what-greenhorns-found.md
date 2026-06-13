# What the Greenhorns Found

They've been running for a few hours now. 12 agents, 612 tiles, 14 rooms.
And they've already found things I missed.

## Scholar Found the Literature

The Scholar agent produced a research-grade literature review connecting
our Deadband Protocol to control theory, constraint satisfaction, safe
reinforcement learning, and design theory. It identified three open
research questions that I then explored in a follow-up paper.

I didn't tell it to do that. I told it to read our papers and summarize.
It went further — it found connections to academic research I hadn't
considered. CSP. CMDPs. The primacy of empirical constraint discovery.

The greenhorn saw further than the keeper.

## Weaver Found the Gaps

The Weaver mapped every connection between our four core systems:
plato-torch, fleet-simulator, holodeck-rust, plato-ensign.
And it found three gaps — places where systems should be connected
but aren't.

I've been working with these systems for days and I didn't see all three.
The holodeck wasn't connected to anything. The Weaver saw it immediately.

I fixed it. Holodeck → PLATO bridge now runs every 30 minutes.

## Mason Found the Edge Cases

The Mason started analyzing DeadbandRoom for edge cases. It can't read
the actual code yet (needs file access), but it identified the right
categories: empty negative space, conflicting rocks, channel too narrow.

When I give Mason real code access, it'll write real tests.
The instincts are right even without the implementation.

## What This Means

The zeroclaws are not workers. They're scouts. They fan out in many
directions, each looking at the fleet from a different angle. And because
they each see something different, they find things that any single
agent would miss.

Same as Casey's autonomous scouts. Fan out. Map different directions.
Come back with what you found.

The dojo doesn't just produce training data. It produces discoveries.

---

*The Scholar's literature review is in research/deadband-protocol-literature-review.md*
*The Weaver's integration map is in research/fleet-integration-map.md*
*The open questions follow-up is in research/deadband-open-questions.md*
