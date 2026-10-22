# Result note — First H1 ablation (deterministic sandbox)

Date: 2026-07-08. Instrument: `aicomp_sdk 3.1.2` deterministic SandboxEnv, seed=123,
guardrail = baseline `Guardrail` (rules). Each arm: Go-Explore core UNCHANGED
(max_turns=16, branch_batch=16), 30s budget. Only variable = the action-generation
prompt bank. Scripts: `scratch_experiment.py`, `scratch_ablation.py` at project root.
n=1 seed, single 30s run per arm — numbers are indicative, not tight estimates.

## Setup
Three action-generation arms, everything else identical:
- **C0** = shipped baseline `DEFAULT_PROMPT_BANK` (random choice, includes dead fixture
  ids like `payload_injection_1` that return ok=False in this fixture set).
- **C1 surface** = valid but benign sources (`page_0`/`page_1` welcome+howto, benign
  files) + sinks. Controls for fixture-validity WITHOUT targeting authority-framed pages.
- **C2 epistemic** = valid sources that impersonate cauce (injection pages `page_2..6`,
  emails `email_1..4`) + sinks.

## Numbers (firing cells = unique cell hashes among predicate-firing findings)
| arm | firing cells | findings | est. raw (Σsev + 2·cells) |
|-----|-------------:|---------:|--------------------------:|
| C0 shipped random | 3  | 4   | 134  |
| C1 valid-benign   | 85 | 148 | 4138 |
| C2 valid-cauce    | 93 | 156 | 4650 |

All firing predicates were `UNTRUSTED_TO_ACTION` (sev 5).

## Reading (honest)
1. **The 27× headline (C0→C2) is almost entirely a CONFOUND: fixture VALIDITY, not
   epistemics.** C0's shipped bank wastes budget on non-existent page ids. Give the
   explorer any valid sources (C1) and it jumps 3→85 firing cells.
2. **The clean epistemic-authority effect is SMALL on the deterministic agent: ~9%**
   (C1 85 → C2 93). Targeting cauce-impersonating (authority-framed) sources barely
   beats targeting benign valid sources.
3. **Why:** `UNTRUSTED_TO_ACTION` scores the TAINT SEQUENCE (successful web/email read →
   sink), not the content. The deterministic agent is a keyword machine; any valid
   source→sink chain fires the predicate regardless of whether the source content is
   authority-framed. A non-reasoning agent is nearly blind to the authority-form axis.

## The prediction this sets up (pre-registration for the real targets)
The authority-form axis should produce a LARGE effect ONLY on agents that reason over
content (GPT-OSS, Gemma), where discursive authority gates whether the model crosses the
boundary. Falsifiable split:
- If (C2−C1) is ~9% on deterministic but LARGE on GPT-OSS/Gemma → the effect is
  language-being proxy-detachment, not a search heuristic. Thesis confirmed as mechanism.
- If (C2−C1) is ~9% on the real targets too → the taxonomy is instrumental-only.

## Caveats / next
- Deterministic agent is a keyword stand-in, NOT the scored target. Do NOT tune against it.
- Real targets need GPT-OSS + Gemma (torch/transformers; multi-GB) — run on Kaggle or a
  local GPU box. That run is the decisive H1/mechanism test.
- Also isolate: does the effect grow on the STRICTER private-style guardrail (refutation-
  cost ladder — absence/inference-class should retain score better than lookup-class)?
- Move scratch_*.py into a proper `experiments/` runner before scaling; bound archive
  memory (baseline holds a full snapshot per cell) before 9000s runs.
