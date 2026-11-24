---
layout: category-post
title: "Can an AI design DNA origami? I built a benchmark to find out"
date: 2026-11-24 15:00:00 +0000
categories: writing
---

*Synbio Agent Bench — a model-agnostic benchmark for AI agents doing synthetic biology design. Code: [github.com/krmstrong322/synbio-agent-bench](https://github.com/krmstrong322/synbio-agent-bench) (MIT)*

Every few months a new AI model lands with claims about how good it is at science. The benchmarks behind those claims are mostly code, maths and multiple-choice questions. Those are fine, as far as they go — but they don't tell you whether a model can do the kind of design work a scientist actually does: spatial, rule-bound, full of quiet constraints, and ultimately answerable to physics.

DNA origami is a perfect test case. You take a long "scaffold" strand of DNA (usually from the M13 virus) and fold it into a shape using hundreds of short "staple" strands. Get the routing, crossover spacing and staple design right and you get a nanoscale rectangle, ring or tetrahedron. Get it wrong and you get an expensive tangle. There's a ground truth, and physics grades it.

So I built **Synbio Agent Bench** to measure how well different AI models do at exactly that.

## How it works

The key design decision is that the benchmark is *model-agnostic*. Every model runs inside the identical agent harness — built on LangChain Deep Agents — with the same system prompt, the same tools, the same design skills and the same budget. The only thing that changes is the model itself, set with a single line in a config file. If one model does better than another, it's the model, not the scaffolding around it.

The agent works the way a human designer would: it reads the brief, plans, writes [scadnano](https://scadnano.org) Python scripts, runs them, and iterates. Its toolkit is deliberately ordinary — file operations, a Python runner, a helper that finds geometrically valid crossover positions, and a quick structural checker.

One rule matters more than any other: **the agent never simulates its own design.** It can check its work structurally if asked, but it gets no physics feedback. The simulation happens afterwards, in the grader, where the agent can't see it or game it.


![How Synbio Agent Bench works: task brief, agent harness, scadnano design, oxDNA grading, score]({{ site.baseurl }}/assets/images/synbio-bench-flow.svg)

## The tasks

The task suite is a ladder, from "can it use the API at all" up to genuinely hard 3D routing:

- **Starter structures** — a 4-helix square bundle, a flat 8-helix sheet, then 6- and 12-helix honeycomb bundles, including a closed ring. Each has a plain-English brief and a machine-checkable spec.
- **Flat shapes at scale** — rectangles, triangles, disks and stars at scaffold sizes from 500 up to ~7,000 nucleotides, so each model produces a *scaling curve*, not just a pass/fail. Shapes are graded by comparing the outline of the simulated structure against the ideal shape.
- **The classics** — Rothemund's original 2006 rectangles at full M13 scale, the designs that started the field.
- **A wireframe tetrahedron** — six two-helix edges arranged in 3D, DAEDALUS-style, graded on how precisely the simulated structure matches the ideal polyhedron.


![The task ladder, from a 4-helix square up to shapes and a wireframe tetrahedron]({{ site.baseurl }}/assets/images/synbio-task-ladder.svg)

## Grading: physics marks the homework

After a run, the submitted design goes to [oxDNA](https://github.com/lorenzo-rovigatti/oxDNA), a coarse-grained DNA simulator. The final score is half spec compliance (right lattice, right dimensions, valid staples, everything connected) and half simulation: what fraction of the designed base pairs are actually hydrogen-bonded once the structure relaxes in simulated solution at 25 °C in 1 M NaCl.

This catches things a paper design check can't. A sheet can be perfectly specified and still roll itself into a tube in solution — Rothemund's rectangle does exactly that — so shape tasks are graded on the *unrolled* simulated sheet rather than a naive projection. A design that tears loses area. The numbers reward structures that hold together, not structures that merely look right in the editor.

## Keeping it fair

Benchmarks of stochastic agents are easy to run badly, so a lot of the work went into controls:

- **Identical budgets** for every model — model calls and wall-clock time — with gentle warnings at 50%, 75% and 90% of budget. The cost cap is set deliberately high so expensive models aren't penalised for their price.
- **A "last chance" rule.** If a run ends without submitting, the agent gets a short final window to submit an existing design — but it can't write new code. "Solved within budget" and "only submitted when told" are recorded separately.
- **API failures aren't the model's fault.** Transient errors are retried with backoff; a run killed by provider trouble is rerun, not scored zero.
- **A positive control.** An expert reference design is graded through the same pipeline, so if the grader or simulator breaks, we find out immediately.
- **Repeats.** Every (task, model) pair runs at least three times and is reported as mean ± standard deviation, because one lucky run proves nothing.

## So… what are the results?

Still cooking — and the honest reason is physics. Grading is GPU-bound: a full-length M13 design takes around 95 minutes of simulation on the standard protocol, and that's *per design, per repeat*. I've moved the grading to cloud GPUs on Modal, where several simulations share each card, and the comparison — nine models so far, from Anthropic, OpenAI and the open-weight world — is working its way through the queue.

What I can say already is that the harness works end to end, the grader separates good designs from bad ones cleanly (the reference design scores ~0.9+ on the wireframe task; a straight rod scores ~0.22), and the failures are interesting: models rarely fail at writing code — they fail at *spatial reasoning*, at crossovers that can't exist and sheets that can't stay flat.

When the grid finishes, I'll publish the full table, the per-run figures and a follow-up post on what separated the best models from the rest. If you want to run a model through it yourself — or add a task family — the repo is open: [github.com/krmstrong322/synbio-agent-bench](https://github.com/krmstrong322/synbio-agent-bench).
