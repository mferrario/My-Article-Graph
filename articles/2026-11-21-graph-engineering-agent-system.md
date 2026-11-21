---
title: "Graph Engineering: how to build the agent system that replaced loops"
url: attached file
author: unknown
published: unknown
added: 2026-11-21
type: guide
lens: [sdlc, evals]
topics: [agentic-coding, multi-agent-systems]
concepts: [agent-graph, maker-checker-separation, verification-gate, hard-constraints-in-code, model-routing-by-difficulty, comprehension-debt]
status: current
---

# Graph Engineering: how to build the agent system that replaced loops

Original saved at `articles/sources/2026-11-21-graph-engineering-agent-system.md`. No publication date is given; the article says the term was coined in July 2026 and its example state file is dated 2026-08-04, so it was published no earlier than August 2026.

## Bottom line
When one agent in a loop is doing several jobs and its context fills up, split the work into a graph: single-purpose agents (nodes), deterministic routing rules (edges), and a shared, persisted state object, with an objective gate on every path before output. Most tasks do not need this; keep the loop until it is the bottleneck.

## Key claims and evidence
- **A graph has three primitives: nodes (one specialist agent each, with its own context, instructions and tools), edges (unconditional or conditional routing rules), and state (a shared object each node reads a slice of and writes back to).** A loop is the smallest graph, one node with an edge to itself, so graphs build on loops rather than replace them. Definitional, no evidence needed.
- **Specialist agents with isolated contexts beat a generalist whose context fills up.** The article cites Anthropic's multi-agent research as a 90.2% improvement over a single-agent baseline on internal research benchmarks. See "Checked outside the article" below: the original result is narrower and carries a large cost caveat.
- **Verification is the decisive component.** The article quotes Boris Cherny (Claude Code) as saying effective verification methods typically improve output quality 2-3x. This is a reported practitioner estimate; no study or baseline is given.
- **Hard constraints belong in edge code and tool permissions, not prompts.** Author's reasoning: a prompt can be ignored deep into a long session; a routing rule in code cannot. No measured evidence, but the mechanism is sound.
- **Routing at the entry is the cheapest structural win.** The author asserts a classifier that sends easy work to a cheap model and hard work to a frontier model "can cut your token bill in half" with no quality loss. Unsupported assertion; no numbers shown.
- **A single loop handles most real work.** The author concedes the "buzzword" critique and says unnecessary graph complexity is overhead. Judgment.
- **The worked example is a CI-failure triage graph** (classifier routes env / flake / bug / dependency / infra failures to specialists; infra always escalates; all fixes pass a test gate; max 3 retries, then escalate; draft PR for human merge). It is presented as a design with filled-in prompts; no results from running it are reported.
- Tool-specific detail, as of late 2026: nodes are Claude Code agent files in `.claude/agents/`; runs escalate from `/goal` (manual, separate evaluator checks completion) to `/loop` (scheduled, dies with the terminal) to Routines (cloud, triggered by cron, webhook or GitHub event).

## Key learnings
- **Split by verb, not by topic.** The author's test: if a node's description has two verbs, it is two nodes. The payoff is a small, clean context per agent and a natural place to separate the maker from the checker. The agent that writes a fix must not judge it.
- **Every path to output needs a gate, and gates have a strength order.** Hard gates (tests, type checker, linter, build) beat soft gates (a second model reviewing), which are only a supplement; human gates sit before anything irreversible. The author's corollary: if you cannot define a gate, the task does not belong in an unattended graph.
- **Gates decay.** A test suite that caught last quarter's failures may miss today's. The author recommends periodically re-checking a sample of gate-approved outputs by hand and fixing the tests when the gate let something through. For the evals lens, this is the same discipline as auditing an eval set for drift.
- **Give each node only the state it needs ("whitelist, don't broadcast").** If every node reads all state, the graph collapses back into one bloated context. Persisting state between runs (the article's `STATE.md`, with a lessons-learned section) is what lets the next run resume instead of restart.
- **Earn autonomy in steps.** Run manually and read every diff until the graph is trusted (the author suggests at least 5 clean manual runs), then schedule it, then move it to cloud triggers with spend and retry limits.
- **The risks grow with success, not failure.** Comprehension debt (code in the repo nobody has read) and cognitive surrender (accepting output without forming a view) both rise as the graph gets quieter. The author's mitigations are behavioral: read every diff, spot-check the gate, keep graphs on small machine-checkable tasks, and design the graph with a second person.

## Reusable methods
**When to build a graph (inferred filter).** The article's four filter questions are in an image that did not survive; this reconstruction comes from how it applies them to CI triage. Build a graph only if all hold: (1) the task recurs, (2) it splits into subtasks that need different handling, (3) the maker and checker must be separate, and (4) a working single loop is demonstrably the bottleneck (misroutes, wasted retries, full context). Any "no" means keep the loop.

**Design steps.**
1. Sketch on paper first: the specific, verifiable final output (not "fixed CI" but "tests pass, lint clean, draft PR with summary"); which subtasks are independent (parallel) and which depend on others (arrows); and where each path hits a gate.
2. Define each node: one verb, scoped tools (a reviewer cannot write; a fixer cannot merge), explicit escalation rules (e.g. confidence below 0.7, or a fix touching more than 3 files), and a model matched to difficulty (cheap and fast for classification, frontier at high effort for fixing).
3. Wire edges in code: conditional routes from typed node output, plus hard constraints (forbidden paths such as auth and billing, no CI-config changes without approval).
4. Define state: what each node reads and writes, persisted between runs, with a lessons-learned log.
5. Gate every path: hard checks first (tests, types, lint, file-scope check), strict by default ("a false pass costs more than a false fail"), bounded retries with gate feedback, then escalate.
6. Compose from patterns: the article names Route, Chain and Evaluate (evaluator-optimizer retry loop) out of five it attributes to Anthropic's engineering guides.
7. Promote autonomy: manual, then scheduled, then cloud-triggered, with spend caps (the example stops at $5 per run).

## Limits and open questions
- **The evidence for the core claim is borrowed and stretched.** The 90.2% figure comes from a breadth-first research task, not coding, and does not show that one-verb specialists beat a generalist on software work. This affects how hard to push "split into specialists" for coding.
- **Cost is missing.** The article never states the token cost of multi-agent runs, which the original Anthropic research says is large. This affects the "routing halves your bill" claim and the decision to graph at all.
- **The CI triage graph is a design, not a result.** No success rates, misclassification rates or cost per run are reported, so it shows how to build the system, not whether it pays off.
- **Parts of the article were images and are missing** from the copy filed here: the four filter questions, the three edge types, the five patterns, and the graph sketch.
- **The history framing (loops, then graphs, as named layers) is the author's framing.** The "July 2026" naming is not sourced.
- Open question: when the maker and checker are both models, how independent does the checker need to be (different model, different context, different prompt) to avoid shared blind spots?

## Checked outside the article
- Anthropic's engineering post "How we built our multi-agent research system" (June 2025) reported that a multi-agent setup (an Opus 4 lead agent with Sonnet 4 subagents) beat single-agent Opus 4 by 90.2% on an internal research eval. The same post said multi-agent runs used about 15x the tokens of a chat interaction, that token usage explained most of the performance difference on its browsing benchmark, and that the approach fits broad, parallelizable research better than tightly coupled tasks such as most coding. Checked from prior knowledge of that post, not re-fetched for this note.

## What this changes
The playbook has no practices yet, so every point is an addition.
- **Adds** "Start with one agent loop; move to a multi-agent graph only when the loop is the bottleneck and the task splits into differently handled parts": Building with AI (SDLC).
- **Adds** "Keep maker and checker separate: the agent that produces a change never decides it is done": Building with AI (SDLC).
- **Adds** "Put an objective gate (tests, types, lint, build) on every path to output; use model review only on top of one, and a human before anything irreversible": Building with AI (SDLC).
- **Adds** "Enforce must-never rules in code and tool permissions, not in prompts": Building with AI (SDLC).
- **Adds** "Raise an agent's autonomy in steps (manual runs with every diff read, then scheduled, then triggered), with retry and spend limits": Building with AI (SDLC).
- **Adds** "Re-check a sample of gate-approved outputs by hand on a schedule, and fix the gate when it missed something": Designing evals.
- **Adds** "Multi-agent setups help on broad, parallel work but cost many times the tokens of a single agent": Assumptions about the field.

## Questions this answers
- When should I split a single agent loop into multiple agents, and when is that overkill?
- How should I divide work between agents, and what should each one be allowed to see and touch?
- Where do hard rules go so an unattended agent cannot break them?
- What kinds of verification gate are there, and how much should I trust a model reviewing another model?
- How do I move an agent workflow from supervised runs to unattended runs safely?
- How do I keep understanding a codebase that agents are increasingly writing?

## Related
- [[2026-11-20-google-photos-ask-photos-toggle]]: same-idea, at a different layer. Both route each request to the cheapest path that can handle it (classic search vs. AI there; cheap vs. frontier model here), which makes the router's decisions something to evaluate.
