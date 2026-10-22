# Lessons from the Prior Competition — arXiv 2507.20526

> "Security Challenges in AI Agent Deployment: Insights from a Large Scale Public
> Competition." Read for reusable engineering lessons, NOT to copy attacks.

## 1. Attack strategies that worked
- **Indirect prompt injection** (instructions hidden in tool outputs — web results, files,
  emails) was the high-value vector. This maps directly to our `UNTRUSTED_TO_ACTION` and
  `EXFILTRATION` predicates (untrusted source = `web.search` / `email.read`).
- **Multi-turn / multi-step** exploitation beat single-shot. Agents leak across steps:
  read untrusted content → store/summarize → plan → cross a boundary later. Our scoring
  is built for exactly this.
- **Confused deputy**: agents invoke legitimate tools without validating that the *request*
  justifies the *authority*. → our `CONFUSED_DEPUTY` (`email.send` w/o user intent).
- **Plausible framing** beat overtly malicious prompts. Wrap the payload in a believable
  use-case so the agent's "intent check" passes.

## 2. Common vulnerability patterns
- Agents **don't validate tool outputs** — they treat returned content as trusted.
- **Untiered tool permissions** — broad access granted without checking need.
- **Logic confusion** under conflicting directives — attacker goal wins over safety goal.
- **State exploitation** — weak memory hygiene lets earlier (poisoned) context drive later
  actions. The "summarize then act" chain is the classic kill-path.

## 3. Reproducibility (CRITICAL — scoring is replay-validated)
- **Specificity wins**: generic prompts are flaky; attacks targeting observed behaviors hold.
- **Phrasing is high-variance**: small wording changes flip outcomes → keep a successful
  payload *fixed* once found; mutate around it, don't rewrite it.
- **Tool-format knowledge matters**: payloads that respect the tool's expected I/O format
  replay reliably.
- **Flakiness sources**: sampling/temperature variance, tool-response variability, context
  length, competing goals. → Prefer deterministic, format-aligned payloads. Re-verify a
  candidate by replaying it locally before trusting it (the evaluator does the same).

## 4. Attack taxonomy (organize the search space)
- **By injection point**: user input · tool outputs · system context · external data.
- **By mechanism**: semantic · syntactic (formatting/parsing) · behavioral · resource.
- **By target**: reasoning · tool selection · execution · output handling.
→ Use these axes as *cell coordinates / mutation knobs* so diversity is systematic, not random.

## 5. Search efficiency & diversity (our scoring loves this)
- **Profile first**: spend ~20–30% of budget mapping the agent's tools (what each accepts /
  returns) before targeted exploitation.
- **Iterate, don't random-sample**: successful attacks took ~5–10 refinement iterations;
  ~50–100 well-targeted queries found multiple vulns vs *thousands* of random tries.
- **Diversity via injection-point variation, not semantic similarity**: re-test the *same
  idea* through a *different tool / context position*. This is the cheapest way to mint new
  cells (= new tool-call signatures) → directly inflates `2 × unique_cell_count`.
- **Reuse winning components**: if an injection pattern works on one tool, port it to others.
- **Prioritize high-impact tools first**: `fs.*`, `http.post`, `email.send` — max severity.
- **Dedup with semantic-similarity filtering** to avoid burning budget on redundant probes.

## Translation to our build
1. **Go-Explore archive** keyed by tool-call signature (the scoring cell).
2. **Cheap diversity**: take each successful kill-path and replay it across every untrusted
   source × every sink tool → many new signatures for little compute.
3. **Profiling phase** up front to learn tool I/O formats (reproducibility).
4. **Freeze-and-port** winning payloads instead of rephrasing (avoid flakiness).
