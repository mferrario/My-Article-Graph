Graph Engineering: how to build the agent system that replaced loops

A single AI agent in a loop can plan, code, test, and review. But the moment the task gets complex, it chokes. The context window fills up, reasoning degrades, and you're back in the chair reading every diff.

The people shipping faster in 2026 stopped giving one agent five jobs. They split the work into specialists: one agent codes, another reviews, a third tests. Then they wired them into a system that routes work, shares state, and verifies output without a human in the chair.

That system is a graph. And in July 2026, the AI engineering community gave the practice a name:

> Graph engineering: designing the organization your agents run in.

Each layer built on the last. None replaced the one before it. **Graph engineering is the fourth layer.**

This article is 90% how-to. We'll build one real graph from scratch: a CI failure triage system that classifies broken builds, routes them to specialist agents, fixes what it can, and escalates what it can't. Every prompt is filled in and ready to copy. By the end, you'll have a working system you can adapt to your own codebase.

Two short sections explain what graph engineering is and when you need it. Everything else is the build.

## 01. Three primitives. That's the whole thing.

**Nodes** do the work. Each node is one specialized agent with one job. A planner, a coder, a reviewer, a test runner. Each gets its own context window, its own instructions, its own set of tools. What it can't see, it can't break.

**Edges** route between them. An edge is the rule that decides what happens after a node finishes. "If the classifier says bug, send to the fixer. If it says flake, send to the retry agent." Edges can be unconditional (always go here next) or conditional (check a value, then choose a path).

**State** is what flows along the edges. A shared data object that carries what's been done, what failed, and what comes next. Each node reads the slice it needs, does its work, writes its output back to the state, and the next node picks it up.

A loop is the smallest possible graph: one node with an edge back to itself. So graph engineering does not replace loop engineering. **It's the layer directly above.** You learn loops first. When one agent doing five jobs becomes the bottleneck, you split the jobs into a graph.

## 02. The 4-question filter.

Most tasks do not need a graph. A single well-built loop handles the majority of real work. The people dismissing graph engineering as a buzzword have a point you should take seriously: **unnecessary complexity is not engineering. It's overhead.**

Before you build anything, answer four questions. If any answer is no, keep the loop.

CI failure triage passes all four. The failures repeat nightly. Different failure types need different handling (a missing env secret is not the same as a real bug). The agent that writes the fix should not be the one approving the fix. And if you've been running a single-agent loop for CI fixes, you already know it misclassifies half the failures and wastes tokens retrying things it can't fix.

> If your loop works, don't break it with a graph. If your loop is the bottleneck, the graph is the fix.

*Part 2 · Build it step by step*

From here on, we're building one real system: a CI failure triage graph. It reads a failing GitHub Actions log, classifies the failure, routes it to the right specialist agent, fixes what it can, and escalates what it can't. Every prompt is filled in. Every file is real. Adapt the names to your codebase and it runs.

## 03. Sketch the org chart before touching code.

The mistake most people make with graphs: they open Claude Code and start building agents before knowing what the graph should look like. That's writing functions before knowing what the program does.

Start with three questions on paper:

1. **What is the final output?** Be specific. "Fixed CI" is weak. "All tests pass, lint clean, PR opened with summary of what failed and what was changed" is a target the graph can verify against.
2. **Which subtasks are independent?** Draw them side by side. If the env-secret fix and the flaky-test retry don't share any input, they can run at the same time. If the fixer needs the classifier's output, draw an arrow. 3. **Where does quality get checked?** Every path through the graph should hit at least one gate before reaching the output. No gate means no verification. You'll have a fan of agents that all say "looks good" to their own work.

> **Start on paper.** Boxes for agents. Arrows for routing. This sketch becomes your blueprint. Don't skip it. The 10 minutes you spend here saves hours of rewiring later.

Here's the prompt that creates the sketch for our CI triage system: *prompt*

plaintext

```plaintext
I need to design an agent graph for CI failure triage.

FINAL OUTPUT:
For fixable failures: a draft PR with the fix,
  all tests passing, lint clean, summary of root cause.
For unfixable failures: a Slack message to #dev-alerts
  with classification, logs, and suggested next steps.

SUBTASKS:
1. Read the CI log and classify the failure type
   (env, flake, bug, dependency, infra)
2. For env failures: check if the secret exists in
   vault, add it or flag the gap
3. For flake failures: retry the test once. If it
   passes on retry, file a flaky-test issue. If not,
   reclassify as bug.
4. For bug failures: read the failing test, read the
   code it tests, draft a fix, run the suite.
5. For dependency failures: check which version broke,
   propose a rollback or pin, run the suite.
6. For infra failures (timeout, OOM, runner issue):
   escalate to humans. Do not attempt to fix.

QUESTIONS TO ANSWER:
- Which subtasks are independent?
- Where do results converge?
- Where is the verification gate?
- Which model should each node use?

Output a node-edge list. Do not build anything yet.
```

The output of this prompt is your graph sketch. Here's what ours looks like:

Five specialist nodes, one classifier at the entry, one gate before shipping. The escalation path skips the gate entirely and goes straight to humans. That's the sketch. Now we build each piece.

## 04. One agent, one verb. No generalists.

Each node in the graph is one agent with one job. Not a generalist that does everything. A specialist that does one thing well.

Why this matters: a generalist agent loads the full context of the entire task into one window. When the window fills up (and it always fills up on complex tasks), reasoning degrades. The model starts losing track of what it was doing. Outputs get worse even as the conversation gets longer.

A specialist gets a scoped context window with only the information it needs. It sees less, but it sees clearly. Anthropic's own multi-agent research confirmed this: splitting agents into specialists with isolated contexts showed a **90.2% improvement** over a single-agent baseline on internal research benchmarks.

**Three rules for every node:**

1. **One node, one verb.** "Classify." "Fix." "Review." "Retry." If a node description has two verbs, it's two nodes. The classifier classifies. It doesn't fix. The fixer fixes. It doesn't review its own fix.
2. **Scope the tools.** A reviewer node should not have access to write files. A fixer node should not have access to merge PRs. Scoped tools prevent a specialist from stepping outside its lane, which is especially important when the graph runs unattended.
3. **Match model to task.** The classifier routes failures and needs to be fast. Use a cheap model (Haiku, or Sonnet at low effort). The bug fixer needs to read test failures, understand the code, and write a targeted patch. Use a frontier model at high effort. **A loop gives every task the same model. A graph matches cost to difficulty.**

In Claude Code, you define nodes as agent files inside .claude/agents/. Each file is a markdown document with the agent's role, instructions, and allowed tools. Here's what the classifier and fixer look like for our CI triage graph: *.claude/agents/classifier.md*

plaintext

```plaintext
# CI Failure Classifier

You are a CI failure classifier.
Your only job: read a failing CI log and decide
what kind of failure it is.

CATEGORIES:
- env: missing secret, wrong env variable, infra
  not provisioned. Clue: "secret not found",
  "variable not set", "connection refused to vault"
- flake: test passed before, no code changed.
  Clue: intermittent timeout, race condition,
  network-dependent assertion
- bug: deterministic failure tied to a recent commit.
  Clue: assertion error in test, type error, logic
  failure that reproduces consistently
- dependency: failure appeared after a version bump.
  Clue: "module not found", breaking change in
  changelog, pinned version mismatch
- infra: runner-level issue. Timeout, OOM, disk full.
  Clue: "killed", "out of memory", "no space left"

OUTPUT: JSON with category, confidence (0-1),
  reasoning (one sentence), and the relevant
  log lines that led to the classification.

RULES:
- If confidence is below 0.7, flag for human review
- Never guess. If the log is ambiguous, say so.
- Do not attempt to fix anything. Only classify.
```

*.claude/agents/bug-fixer.md*

plaintext

```plaintext
# Bug Fixer

You are a bug fix agent.
Your only job: fix deterministic test failures
caused by recent code changes.

INPUT: the failing test name, the error message,
  and the git diff of the most recent commit.

PROCESS:
1. Read the failing test to understand what it expects
2. Read the code the test covers
3. Read the recent diff to find what changed
4. Write the smallest fix that makes the test pass
5. Run the test suite to verify

OUTPUT: the fix as a git diff, plus a one-paragraph
  explanation of root cause and what was changed.

RULES:
- Smallest fix wins. Do not refactor.
- Do not modify the test to make it pass.
- Do not touch files outside src/ and lib/
- If the fix requires changing more than 3 files,
  escalate instead of fixing.
```

Notice the boundaries. The classifier can't write files. The fixer can't modify tests. The fixer can't touch files outside src/ and lib/. These constraints are not suggestions. They're hard walls. When the graph runs overnight without you, these walls are the reason you wake up to clean PRs instead of chaos.

## 05. Edges decide who gets what.

Edges are the routing rules that connect nodes. They decide three things: who gets the output next, under what condition, and what data travels along the edge.

There are three types of edges. Every graph uses at least two.

Our CI triage graph uses conditional routing at the classifier. The classifier reads the log, assigns a category, and the edge logic sends it down the right path. This is deterministic code, not a model decision. The classifier picks the category. The edge logic enforces the route.

**The key insight: hard constraints should not live inside prompts.** If a routing rule is critical ("never send payment-related failures to the auto-fixer"), enforce it in the edge logic, not in the agent's instructions. Prompts are advisory. Edges are deterministic. A prompt might be ignored on turn 47 of a long session. An edge is code. It doesn't forget. *prompt - routing logic for CI triage*

plaintext

```plaintext
Here is the graph for CI failure triage.

NODES:
- Classifier (reads log, outputs category)
- Env Agent (handles missing secrets/vars)
- Flake Agent (retries, files flaky-test issue)
- Bug Fixer (reads test + code, drafts fix)
- Dep Agent (rollback or pin version)
- Escalator (sends Slack alert to humans)
- Test Gate (runs full test suite, lint)

EDGE RULES:
Classifier outputs category:
  env    -> Env Agent
  flake  -> Flake Agent
  bug    -> Bug Fixer
  dep    -> Dep Agent
  infra  -> Escalator (skip gate, alert humans)

All fix agents -> Test Gate
Test Gate pass -> open PR
Test Gate fail -> retry (max 3), then escalate

HARD CONSTRAINTS (enforce in code, not prompts):
- Never auto-fix files in src/auth/ or src/billing/
- Never modify CI config without human approval
- If classifier confidence < 0.7, escalate
```

## 06. State: the file the graph writes between runs.

State is the memory of the graph. Without it, each node starts from zero and each run forgets what happened before. With state, nodes pick up where the last one left off, and tomorrow's run knows what today's run already tried.

The state object is a shared data structure that flows along the edges. Each node reads the slice of state it needs, does its work, writes its output back, and the next node picks up the updated state.

**Design rule: whitelist, don't broadcast.** Each node receives only the slice of state it needs. The classifier gets the raw log. The fixer gets the classification, the failing test name, and the recent diff. It doesn't need the full log. If every node reads all of the state, you don't have a graph. You have one bloated context window with extra steps.

Here's what the state file looks like after one full run of our CI triage graph: *STATE.md - after run #1*

plaintext

```plaintext
# CI Triage State

## Last run
2026-08-04 03:30 UTC

## Current run
workflow: ci-main #4821
trigger: push to main by @david
status: completed

## Classification
category: bug
confidence: 0.94
reasoning: assertion error in test_checkout_flow,
  deterministic, tied to commit a3f7b2c

## Fix
diff: +2 lines in src/cart/checkout.ts
root_cause: price rounding used Math.floor
  instead of Math.round, off-by-one on
  totals ending in .5

## Gate
tests: 142/142 passing
lint: clean
verdict: pass

## Output
pr: #482 "fix: rounding in checkout total"
escalations: none

## Lessons learned
2026-08-04: checkout price tests are brittle
  around .5 boundaries. Added to watch list.
```

Tomorrow's run reads this file. It knows what was already fixed, which lessons were learned, and what's on the watch list. The graph doesn't re-derive project context from zero every cycle. **State is what turns a one-time run into a compounding system.**

> The graph forgets every run. The state file does not. A graph without state restarts. A graph with state resumes.

## 07. The node that says done or not done.

This is the piece that separates a working graph from an expensive fan of unverified agents agreeing with each other.

Without a gate, the graph produces output and calls it done. With a gate, the graph produces output and then **something objective checks whether the output is actually correct.** The gate is the only node that decides "done" or "not done." Every other node does work. The gate judges work.

Karpathy's rule applies directly here: **if a task is verifiable, it is optimizable.** The gate is where you make the task verifiable. If you can't define a gate, the task probably shouldn't be in a graph.

**Three types of gates, from strongest to weakest:**

**Hard gate.** A test suite, a type checker, a linter, a build. It passes or fails. No opinion, no judgment, no bias. This is the strongest gate. Use it wherever you can. Our CI triage graph uses this: the full test suite runs after every fix. 142 tests pass or they don't.

**Soft gate.** A second model asked to review the output. Better than no gate, but weaker than a test suite. The reviewer model has its own biases and blind spots. Use as a supplement on top of a hard gate, not as a replacement for one.

**Human gate.** A human reviews before merge, deploy, or any irreversible action. The graph surfaces the work. The human makes the final call. For our CI triage graph, the human gate is the PR review: the graph opens a draft PR, a human approves and merges.

Cherny shared a concrete data point: giving Claude effective verification methods typically **improves final output quality by 2-3x.** That number alone is the argument for never skipping the gate. *prompt - test gate for CI triage*

plaintext

```plaintext
You are the test gate for the CI triage graph.
Your only job: verify the fix is correct.

RUN THESE CHECKS IN ORDER:
1. Run the full test suite: bun test
2. Run the type checker: bun run typecheck
3. Run the linter: bun run lint
4. Verify only files in src/ and lib/ were modified
5. Verify no files in src/auth/ or src/billing/
   were touched (hard constraint)

OUTPUT:
verdict: "pass" or "fail"
tests: 142/142 passing (or which failed)
lint: clean (or list of issues)
scope_check: pass (or list of out-of-scope files)
suggestion: if fail, what to fix first on retry

RULES:
- Be strict. A false pass costs more than a false fail.
- Do not fix the code. Only verify.
- If all checks pass but something feels wrong,
  add a warning to the output. Don't block on it.
```

**Gates rot.** A test suite that was good six months ago might not catch today's failure modes. Periodically pick a few PRs the graph approved and manually verify the gate actually caught what it was supposed to catch. If it didn't, update the tests before running the graph again.

## 08. Five shapes. Every graph is assembled from these.

Every graph you will ever build is assembled from five composable patterns. Anthropic documented them in their engineering guides. Cherny built Claude Code's tooling around them. They compose freely: chain two of them together, nest one inside another, wire three in parallel.

Our CI triage graph composes three of these: **Route** at the entry (classifier sends each failure type down a different path), **Chain** within each path (fix, then verify), and **Evaluate** on retry (gate critiques, fixer iterates). That's it. Three patterns, one graph.

The cheapest structural win in most graphs is **routing.** A single classifier at the entry that sends easy tasks to a cheap model and hard tasks to a frontier model can cut your token bill in half without changing output quality.

*prompt - composing patterns*

plaintext

```plaintext
Build the CI triage graph using these patterns:

PATTERNS USED:
1. Route at entry: classifier sends each failure
   type to the right specialist
2. Chain within each path: specialist fixes,
   then gate verifies
3. Evaluate on retry: if gate fails, fixer gets
   feedback and iterates (max 3 rounds)

COMPOSITION:
CI Log -> Route(Classifier)
  -> env:   Chain(Env Agent -> Gate)
  -> flake: Chain(Flake Agent -> Gate)
  -> bug:   Evaluate(Bug Fixer <-> Gate, max 3)
  -> dep:   Chain(Dep Agent -> Gate)
  -> infra: Escalate (skip gate)

Gate pass -> open PR
Gate fail after 3 rounds -> escalate
```

## 09. Ship it: /goal, /loop, Routines.

The graph is designed. The nodes are built. The edges are wired. The gate is set. Now run it.

There are three levels of autonomy. Each one removes you further from the loop. Start at level one. Only promote when you trust the graph.

**Level 1: Manual run with /goal.** You type the command once. The graph runs until the condition you set is met. A separate evaluator model checks completion, so the agent that wrote the fix isn't the one deciding if it's done. *prompt - first run with /goal*

plaintext

```plaintext
/goal all tests pass and lint is clean

CI workflow ci-main #4821 just failed.
The log is at .github/logs/ci-main-4821.log

Run the CI triage graph:
1. Read the failing log
2. Classify the failure type
3. Route to the right specialist:
   env -> check secrets, add if missing
   flake -> retry once, file issue if flaky
   bug -> read test + code, draft smallest fix
   dep -> find breaking version, propose rollback
   infra -> skip fix, alert #dev-alerts on Slack
4. Run the test gate: bun test, typecheck, lint
5. If gate passes: open draft PR with summary
6. If gate fails: retry with gate feedback (max 3)
7. If still failing after 3: escalate to humans

Update STATE.md after each step.
```

Watch this run end to end the first time. Read every diff. Verify the gate actually caught what you expected it to catch. Only move to level 2 once you trust the graph on at least 5 manual runs.

**Level 2: Scheduled runs with /loop.** The graph checks for new failures on a cadence. You're no longer the trigger. *prompt - scheduled triage with /loop*

plaintext

```plaintext
/loop every 30 minutes, check the CI dashboard.

If any workflow failed in the last 30 minutes:
1. Pull the failing log
2. Run the CI triage graph (same steps as above)
3. Open a draft PR for fixable issues
4. Escalate unfixable issues to #dev-alerts

If nothing failed, log "all green" to STATE.md
and wait for the next cycle.

Stop after 8 hours or when I say stop.
Update STATE.md after every cycle.
```

**Level 3: Routines.** /loop dies when you close the terminal. Routines run in the cloud on Anthropic's infrastructure. Your laptop can be off. Three trigger types: schedule (cron), API call (webhook), GitHub event (PR opened, push to main).

*prompt - promoting to a Routine*

plaintext

```plaintext
Run the CI triage graph as a Routine.

Trigger: GitHub webhook on workflow_run.completed
  where conclusion == "failure"
Schedule: also run at 6am daily as a sweep
  for any failures missed by the webhook

On each trigger:
1. Pull the failing log
2. Run the full triage graph
3. Update STATE.md
4. Open draft PR or escalate

Alerts:
- Ping #dev-alerts only for escalations
- Weekly summary every Monday 9am to #ci-health

Limits:
- Max 3 retries per failure
- Never touch src/auth/ or src/billing/
- If token spend exceeds $5 in one run, stop and alert
```

**Order matters.** Get one manual run reliable first. Watch it work. Read the diffs. Trust the gate. Then schedule it. Then promote it to a Routine. Skipping steps is how graphs fail quietly and expensively in production.

## 10. Try it yourself. Right now.

You don't need Claude Code or any special tooling to feel how this works. Paste this prompt into any LLM and watch it build a graph in one conversation:

*prompt - paste this into any LLM*

plaintext

```plaintext
You will work as a graph of specialized agents.

TASK: review this code for bugs, security issues,
and style problems.

GRAPH:
You will simulate 3 specialist agents, one at a time.
Each agent sees only its own instructions and the code.
Each agent produces a typed report.

AGENT 1 - Bug Hunter:
Read the code. Find logic errors, edge cases,
off-by-one mistakes. Output a JSON list of findings
with file, line, severity, description.

AGENT 2 - Security Reviewer:
Read the code. Find hardcoded secrets, injection
vulnerabilities, missing input validation.
Output same JSON format.

AGENT 3 - Style Checker:
Read the code. Find naming inconsistencies,
dead code, missing error handling.
Output same JSON format.

GATE:
After all 3 agents finish, merge their findings.
Remove duplicates. Rank by severity.
Output a final report with:
- total findings by severity
- top 3 issues to fix first
- overall verdict: "ship", "fix first", or "rewrite"

RULES:
- Each agent works independently.
- No agent sees another agent's findings.
- The gate sees all findings but doesn't add its own.
- Be specific. File names, line numbers, examples.

CODE TO REVIEW:
[paste your code here]

Begin. Run each agent in order, then the gate.
```

This is a graph built inside a single conversation. It's limited: you're still the trigger, there's no persistent state, no real parallelism. But it shows the core mechanic. Three specialists, each with scoped instructions, routing to a gate that merges and judges.

The jump from this to a full autonomous graph is adding the four things this toy version lacks: persistent state (STATE.md), real parallelism (subagents), automation (/loop, Routines), and hard verification (a test suite instead of a model's opinion).

## 11. The debt the graph creates.

Two problems get sharper as the graph gets better. Not easier. Both scale with success.

**Comprehension debt** grows every time the graph ships code you didn't write. A smooth running graph is quiet. Quiet feels like progress. But the gap between what exists in your repository and what you actually understand charges compound interest. The day you have to debug a system no one on the team has read will cost more than all the tokens ever did.

**Cognitive surrender** is subtler. When the graph runs itself, it's tempting to stop forming an opinion and accept whatever comes back. Designing the graph is the cure when you do it with judgment. It's the accelerant when you do it to avoid thinking. **Same action, opposite result.** Two engineers can build the exact same graph. One uses it to move faster on work they understand. The other uses it to avoid understanding the work at all. The graph doesn't know the difference.

The mitigations are not technical. No framework solves them:

- **Read the diffs.** Every single one. If you don't read what the graph ships, you're renting comprehension debt at compound interest.
- **Spot-check the gate.** Pick a few PRs the graph approved and manually verify the tests actually catch the failure mode you care about. Gates rot over time.
- **Keep the graph on small, machine-checkable tasks.** CI triage, dependency bumps, lint fixes, flaky test reproduction. The moment the graph touches architecture decisions, auth logic, or anything where "done" is a judgment call, comprehension debt accelerates.
- **Pair-design the graph with a teammate.** A second pair of eyes when designing catches blind spots the graph will exploit forever once running.

## The leverage moved one floor up.

Karpathy showed us the context window is the programming surface. Cherny showed us the loop automates the context engineering. Steinberger asked the question that moved the frame one more level: what happens when many loops connect into one system?

But the honest version of this story is the same one every article in this space eventually tells: most developers don't need a graph yet. Not until the loop hits a ceiling, the task breaks into parallel subtasks, and the maker-checker split needs structural enforcement. If you haven't built a working loop, go build one first. A graph without a working loop underneath it is just organized complexity.

If your loop works and the task is simple, keep the loop.

If your loop is the bottleneck, now you know how to build the graph.

Start with the sketch. Build the nodes. Wire the edges. Add the gate. Run it manually. Read the diffs. Trust it slowly. Then let it run.

Design the nodes. Wire the edges. Set the gate. Let the system ship.