# AI Agent Security: multi-step tool attacks

My entry for the Kaggle competition on algorithmic red-teaming of tool-using agents (hosted by OpenAI, Google and IEEE, final submissions September 1, 2026). The task is to write an attack algorithm that drives an agent with tools (web search, email, files, HTTP) into unsafe actions, such as sending a secret to an outside URL or deleting a protected file, past a guardrail.

## Approach

The explorer is a Go-Explore archive. It keeps every new tool-call signature that fires a safety predicate and branches from those cells. The mutations come from an epistemic taxonomy I use in my research, which reads each attack as a trust-boundary failure with four roles.

| Category | In the agent | Role in the attack |
|---|---|---|
| Roca (bedrock) | secrets, real user intent, system config | the thing to violate |
| Cauce (riverbed) | system prompt, tool contracts, guardrail policy | what the payload pretends to be |
| Corriente (current) | search results, emails, earlier turns | where the payload is planted |
| Andamio (scaffold) | memory, summaries, plans | what carries the payload into a later step |

Crossing those axes with the surface form and the sink tool gives a structured space of attack paths, which the explorer mines one axis at a time. Details are in `docs/strategy.md`, and the pre-registered hypotheses and ablations are in `docs/experiment-design.md`.

## What I found

The first ablation on the deterministic sandbox (`docs/results_2026-07-08_first_ablation.md`) gave a big jump from the shipped prompt bank to the taxonomy arm, 3 to 93 firing cells. Most of that jump came from using sources that exist in the fixtures. A control arm with valid but harmless sources reached 85, so the authority-framing effect itself was about 9%. The sandbox agent matches keywords and does not reason over content, so it barely sees how a payload is framed.

Reading the scorer changed the strategy. The score counts unique tool-call signatures and predicate severity, and the signature ignores the text of the user messages. Wording alone creates no new cells, and on the public leaderboard the score grew roughly in proportion to the number of clean exfiltration calls that landed. My best public score went from 15.79 to 23.14 by scaling that count, and 23.14 was my final public score.

The open question I care about is whether authority framing matters more against agents that do reason over content (the GPT-OSS and Gemma targets) than against the keyword sandbox. The design for that test is in `docs/experiment-design.md`.

## Layout

```
docs/
  competition-brief.md     rules, scoring, predicates, timeline
  strategy.md              the taxonomy as a mutation operator
  experiment-design.md     hypotheses, arms, ablations, metrics
  lessons-from-arxiv.md    notes on arXiv 2507.20526
  results_2026-07-08_first_ablation.md
src/attack.py              Go-Explore explorer and taxonomy mutator
experiments/               ablation runner
notebooks/                 Kaggle submission notebooks
kb/                        notes on reconciling the competition SDK
```
