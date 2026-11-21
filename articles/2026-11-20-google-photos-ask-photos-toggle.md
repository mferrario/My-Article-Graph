---
title: "Google Photos Now Lets Users Disable 'Ask Photos' Search Experience"
url: https://www.thurrott.com/a-i/333542/google-photos-now-lets-users-disable-ask-photos-search-experience
author: Laurent Giret
published: 2026-03-10
added: 2026-11-20
type: report
lens: [industry, evals]
topics: [ai-search, consumer-ai-adoption]
concepts: [ai-opt-out-control, query-routed-hybrid-results, latency-as-adoption-blocker, head-query-quality-tuning]
status: current
---

# Google Photos Now Lets Users Disable 'Ask Photos' Search Experience

## Bottom line
In March 2026 Google Photos backed off making its generative "Ask Photos" the default search: it added a toggle to turn Ask Photos results off, and redesigned search so Google picks classic or AI results per query while users can switch views. A flagship consumer AI feature is being repositioned from replacement to option, after earlier latency problems and user pushback.

## Key claims and evidence
- **Google Photos is adding a toggle to disable Ask Photos results and return to classic search (as of 2026-03-10).** Source: an X thread by Shimrit Ben-Yair, VP and GM of Google Photos, dated 2026-03-09, embedded in the article. Google said it is "starting to roll out" the change; the article gives no platforms, dates or rollout share, and no instructions for finding the toggle.
- **The stated reason is user feedback asking for control.** Ben-Yair said users wanted "more control over the type of results you see." The article reads this as some users preferring search without AI. This is Google's framing; no usage or satisfaction data is given.
- **The new design routes each query to classic or AI results.** Per Ben-Yair, search will lead with whichever result type best fits the query (fast classic search or "intelligent" Ask Photos), and users can always switch. How the routing decides is not described.
- **Google tuned quality on popular queries.** Ben-Yair named "turtles," "purple dress" and "lipstick" as searches that now work better, based on feedback. These are anecdotal examples, not measured results.
- **Ask Photos has a history of rollout trouble.** Per earlier Thurrott reporting linked from the article: Google paused the rollout to fix latency, and the feature left early preview in June 2025. The article gives no detail on the pause.
- A smaller addition: tapping a search result can now surface more photos from that date.

## Key learnings
- **An AI search that replaces a fast, familiar one gets judged against it on speed and precision, not on capability.** Classic photo search answers simple lookups ("turtles") instantly; a generative layer that is slower or worse on those queries reads as a regression even if it handles harder ones. Inference: this is why Google's fix was routing, not a better model.
- **Query routing plus a visible escape hatch is the design pattern Google settled on.** Rather than one experience for every query, a router picks the cheaper deterministic path where it suffices and the AI path where it adds value, and the user can override. Inference: the router's decisions become a core thing to evaluate.
- **Head queries need their own quality bar.** Google's fixes targeted the most popular searches. Inference: an eval set for an AI search feature should over-weight the frequent, simple queries users already expect to work, alongside the hard natural-language ones the AI was built for.
- **An opt-out toggle is both a user control and a public signal.** Inference: shipping one concedes that the AI default did not win on its own, and the share of users who flip it is a direct adoption metric.

## Limits and open questions
- No numbers: no opt-out rate, latency figures, or quality metrics. Every claim about why rests on Google's statements and the author's framing; this limits how far the "users rejected the AI default" reading can go.
- The routing logic and how often each path wins are not described, so it is unclear whether AI results remain the default for most queries.
- Rollout scope and platforms are unknown as of the article date.
- Short news report (about 2 minutes); the earlier latency pause is cited but not explained.

## What this changes
- **Adds** to Assumptions about the field: consumer AI features that replace an existing fast workflow face pushback on latency and head-query precision, and vendors (here Google, 2026-03) answer with per-query routing plus a user opt-out rather than forcing the AI default.
- **Adds** to Designing evals: for an AI layer over existing search, evaluate the router (did it pick classic vs. AI correctly) and over-weight popular simple queries against the classic baseline, not just AI-only answer quality.

## Questions this answers
- How did Google Photos respond to pushback on Ask Photos?
- Should an AI search feature replace classic search or sit beside it?
- What should an eval set for an AI search layer weight, given users already have a fast baseline?
- What signals tell me a consumer AI default is not landing?
- What did Ask Photos' rollout run into before 2026?

## Related
- [[2026-11-21-graph-engineering-agent-system]]: same-idea, at a different layer. Both route each request to the cheapest path that can handle it (classic search vs. AI here; cheap vs. frontier model there).
