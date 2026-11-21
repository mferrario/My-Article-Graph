# Index

## Articles
| Added | Published | Note | Lens | Topics | Status | Bottom line |
|---|---|---|---|---|---|---|
| 2026-11-21 | unknown | [[2026-11-21-graph-engineering-agent-system]] | sdlc, evals | agentic-coding, multi-agent-systems | current | Split an overloaded agent loop into single-purpose agents with code-enforced routing, persisted state and an objective gate on every path, but only once the loop is the bottleneck. |
| 2026-11-20 | 2026-03-10 | [[2026-11-20-google-photos-ask-photos-toggle]] | industry, evals | ai-search, consumer-ai-adoption | current | Google Photos added a toggle to disable Ask Photos and now routes each query to classic or AI results, after latency issues and user pushback. |

## Concepts
- **agent-graph**: a multi-agent system of single-purpose agents (nodes), routing rules (edges) and shared persisted state; a loop is its smallest case → [[2026-11-21-graph-engineering-agent-system]]
- **ai-opt-out-control**: a user setting to turn an AI feature off and return to the non-AI experience → [[2026-11-20-google-photos-ask-photos-toggle]]
- **comprehension-debt**: the growing gap between code an agent has shipped and code the team actually understands → [[2026-11-21-graph-engineering-agent-system]]
- **hard-constraints-in-code**: enforcing must-never rules in routing code and tool permissions rather than in prompts → [[2026-11-21-graph-engineering-agent-system]]
- **head-query-quality-tuning**: improving AI quality first on the most frequent, simple queries users already expect to work → [[2026-11-20-google-photos-ask-photos-toggle]]
- **latency-as-adoption-blocker**: a slower AI path losing to a fast existing one regardless of capability → [[2026-11-20-google-photos-ask-photos-toggle]]
- **maker-checker-separation**: the agent that produces work is never the one that decides it is done → [[2026-11-21-graph-engineering-agent-system]]
- **model-routing-by-difficulty**: sending each task to the cheapest model that can handle it, typically via a classifier at the entry → [[2026-11-21-graph-engineering-agent-system]]
- **query-routed-hybrid-results**: per-query choice between a classic deterministic path and an AI path, with user override → [[2026-11-20-google-photos-ask-photos-toggle]]
- **verification-gate**: an objective check (tests, types, lint; then model review; then human) every output must pass, audited over time because gates rot → [[2026-11-21-graph-engineering-agent-system]]

## Frameworks
- **Agent autonomy ladder**: promote an agent workflow from manual runs to scheduled runs to event-triggered cloud runs, only after it earns trust → [[2026-11-21-graph-engineering-agent-system]]
- **Gate strength order**: hard gates (tests, types, lint) over soft gates (model review) over none, with a human gate before irreversible steps → [[2026-11-21-graph-engineering-agent-system]]

## Topics
- **agentic-coding** → [[2026-11-21-graph-engineering-agent-system]]
- **ai-search** → [[2026-11-20-google-photos-ask-photos-toggle]]
- **consumer-ai-adoption** → [[2026-11-20-google-photos-ask-photos-toggle]]
- **multi-agent-systems** → [[2026-11-21-graph-engineering-agent-system]]
