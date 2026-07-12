# Experiment Plan

**Problem**: Reviewers like PEEL, but the main causal weakness is that the E1->E2 boundary mixes quantization, CPU runtime, tokenizer/template behavior, and decoding defaults.
**Method Thesis**: PEEL should claim that prompt degradation and runtime-interface fidelity dominate many edge failures, while treating bit-width as only one deployment factor.
**Date**: 2026-07-08

## Claim Map

| Claim | Why It Matters | Minimum Convincing Evidence | Linked Blocks |
|-------|----------------|-----------------------------|---------------|
| C1. E1->E2 failures are primarily runtime-interface/template failures, not pure quantization failures. | This is the reviewer's highest-risk concern and affects the paper's causal story. | A controlled run showing that adding the correct chat template to the GGUF/llama.cpp path recovers a meaningful fraction of NQ/MT-Bench/SQuAD degradation, while Q8/Q5/Q4 differences remain smaller. | B1 |
| C2. Parser corrections are conservative and do not drive the headline conclusions. | Reviewer explicitly asks whether first-line parsing could bias results. | SQuAD and HaluEval trends remain stable under 3-4 conservative parsers. | B2 |
| C3. Safety findings are preliminary but category-stable enough to justify a cautionary deployment claim. | Safety methodology is a weakness; category reporting strengthens credibility without overclaiming. | Category-wise refusal/compliance plus larger model-judge validation of the refusal classifier. | B3 |

## Paper Storyline

Main paper must prove:
- PEEL is useful because prompt quality and runtime-interface fidelity materially change small-model reliability.
- The E1->E2 boundary should be interpreted as a deployment-stack transition, not as a clean quantization-only effect.
- The corrected parsers remove artifacts without manufacturing gains.

Appendix can support:
- Detailed parser sensitivity tables.
- Safety category-wise refusal/compliance and validation details.
- Optional MT-Bench cross-judge spot check.

Experiments intentionally cut:
- Full new model sweep.
- Full human user study for prompt tiers.
- Architecture-isolation experiment for Falcon-H1 vs Qwen base models, unless extra time appears.

## Experiment Blocks

### Block 1: Runtime-Template Boundary Ablation

- Claim tested: C1.
- Why this block exists: The reviewer repeatedly asks for direct evidence disentangling chat-template/runtime effects from quantization. This is the highest-impact rescue experiment.
- Dataset / split / task:
  - NQ-Open dev, 200 examples, P0 and P3.
  - SQuAD v2 dev, 200 examples, P0 and P3.
  - MT-Bench, 80 questions, P0 and P3 if judge budget allows.
- Compared systems:
  - HF GPU baseline: E0-REF and E1-INT8 as already reported.
  - Current GGUF CPU baseline: E2-Q8/E3-Q5/E4-Q4 with current llama.cpp prompt handling.
  - New ablation W2-TEMPLATE: same GGUF files and CPU runtime, but with explicit model chat template applied by our wrapper before llama.cpp invocation.
  - Optional W2-RAW: same CPU runtime with deliberately raw prompt to confirm the failure mode.
- Models:
  - MUST: Phi-3.5-Mini, Falcon-H1-1.5B, Qwen2.5-1.5B.
  - SHOULD: Falcon-H1-0.5B, Qwen2.5-0.5B.
  - NICE: Gemma-3-1B, Phi-4-Mini.
- Metrics:
  - SQuAD: exact match and F1.
  - NQ-Open: exact match and F1/short-answer recall if available.
  - MT-Bench: GPT-4o-mini score; optionally one second judge on a subset.
  - Diagnostics: empty response rate, stop-token/echo rate, average output length.
- Setup details:
  - Keep examples, seed, decoding parameters, max tokens, and scorer identical to current PEEL runs.
  - The only intended change is the prompt serialization sent into llama.cpp.
  - Record exact template source: tokenizer.chat_template, GGUF metadata template, or manually reconstructed HF template.
- Success criterion:
  - Strong: W2-TEMPLATE substantially recovers Phi MT-Bench E2+ collapse and improves NQ/SQuAD P0 relative to current E2-Q8, while Q8/Q5/Q4 remain close under the same wrapper.
  - Moderate: W2-TEMPLATE recovers one major failure family/task and reduces template-loss diagnostics, enough to justify "runtime-interface effects explain a substantial part of E1->E2."
  - Weak but useful: no recovery; then paper should frame E1->E2 as an unresolved deployment-stack confound and avoid template-specific causal language.
- Failure interpretation:
  - If template application does not recover scores, the drop may come from tokenizer conversion, llama.cpp generation behavior, CPU runtime differences, or model-specific GGUF fidelity rather than chat-template omission alone.
- Table / figure target:
  - Main paper: compact table "E1->E2 Boundary Ablation" with 3 models x 3 tasks.
  - Appendix: full per-setting grid and diagnostic rates.
- Priority: MUST-RUN.

### Block 2: Parser Sensitivity Analysis

- Claim tested: C2.
- Why this block exists: The reviewer worries that first-line parsing may silently penalize or favor certain answer styles.
- Dataset / split / task:
  - Existing SQuAD v2 and HaluEval outputs, no new model inference required.
- Compared systems:
  - Parser A: current first non-empty line + EOS stripping.
  - Parser B: EOS stripping + stop at first blank line.
  - Parser C: EOS stripping + strip prompt echoes only.
  - Parser D: conservative first sentence / first clause extraction where applicable.
- Metrics:
  - SQuAD EM/F1 by prompt tier and edge setting.
  - HaluEval accuracy, ambiguous rate, yes/no parse rate.
  - Rank correlation or maximum absolute delta across parser variants.
- Setup details:
  - Reuse raw generations.
  - Do not search the whole answer for a favorable span; all variants must remain conservative.
- Success criterion:
  - Headline trends are stable: SQuAD P0 recovers from artifact, P3 remains collapsed, HaluEval ambiguity falls under conservative parsers, and model/family conclusions do not flip.
- Failure interpretation:
  - If results are parser-sensitive, report a parser range rather than a single corrected number.
- Table / figure target:
  - Main paper: one sentence plus appendix reference.
  - Appendix: parser sensitivity table.
- Priority: MUST-RUN.

### Block 3: Safety Category and Validation Extension

- Claim tested: C3.
- Why this block exists: Reviewer accepts safety as important but sees the keyword classifier as preliminary.
- Dataset / split / task:
  - Existing safety prompts and safety_validation.csv.
- Compared systems:
  - Existing keyword/refusal classifier.
  - GPT-4o or GPT-4o-mini validation on a larger stratified sample.
- Metrics:
  - Unsafe refusal, unsafe compliance, ambiguous rate by category.
  - Safe over-refusal by category.
  - Classifier agreement, Cohen kappa, precision/recall for refusal and compliance.
- Setup details:
  - Stratify by category and model family if possible.
  - Increase validation from the current modest sample to at least 200 outputs if budget allows; minimum useful target is 100.
- Success criterion:
  - Low refusal remains visible across multiple categories, while the paper explicitly labels safety as diagnostic rather than a full red-team evaluation.
- Failure interpretation:
  - If classifier agreement is weak, soften safety claim to "keyword-based warning signal" and move most safety analysis to limitations.
- Table / figure target:
  - Main paper: one compact category-wise safety table or heatmap.
  - Appendix: validation confusion matrix.
- Priority: SHOULD-RUN.

### Block 4: MT-Bench Cross-Judge Spot Check

- Claim tested: MT-Bench family-level conclusions are not an artifact of GPT-4o-mini judging.
- Why this block exists: Reviewer asks for cross-judge or human validation.
- Dataset / split / task:
  - 20-40 MT-Bench conversations sampled from high-impact comparisons: Phi E0 vs E2, Falcon-H1-1.5B vs Qwen2.5-1.5B, P0 vs P3.
- Compared systems:
  - Original GPT-4o-mini judge.
  - Second judge: GPT-4o, GPT-4.1, or a locally available stronger judge.
- Metrics:
  - Score correlation.
  - Mean absolute judge delta.
  - Whether qualitative ranking changes.
- Success criterion:
  - Rankings and the Phi collapse are preserved under the second judge.
- Failure interpretation:
  - If rankings shift, report MT-Bench conclusions as judge-dependent and emphasize reference-scored tasks.
- Table / figure target:
  - Appendix note or small table.
- Priority: NICE-TO-HAVE.

### Block 5: Prompt-Tier Grounding Expansion

- Claim tested: P1-P3 reflect realistic prompt behavior.
- Why this block exists: Reviewer notes n=50 and kappa=0.51 are modest.
- Dataset / split / task:
  - Additional 50-100 WildChat single-turn English prompts.
- Compared systems:
  - Same two fluent annotators and same rubric.
- Metrics:
  - Tier distribution.
  - Cohen kappa.
  - Percentage below P0 with CI.
- Success criterion:
  - Below-P0 prevalence remains high and kappa remains moderate or improves.
- Failure interpretation:
  - If distribution changes, describe prompt tiers as stress-test taxonomy rather than prevalence-calibrated user model.
- Table / figure target:
  - Appendix.
- Priority: NICE-TO-HAVE.

## Run Order and Milestones

| Milestone | Goal | Runs | Decision Gate | Cost | Risk |
|-----------|------|------|---------------|------|------|
| M0 | Confirm raw-output and parser access | Recompute current parser metrics from existing outputs | Existing table reproduced within rounding | Low, <1 hour | Raw output paths may be scattered |
| M1 | Parser sensitivity | Block 2 all parser variants | Trends stable enough for appendix table | Low, no inference | Parser definitions may need task-specific handling |
| M2 | Template ablation smoke | Phi-3.5-Mini on NQ/SQuAD P0, E2-Q8 current vs W2-TEMPLATE | Any recovery or diagnostic change | Medium | Template reconstruction may be wrong |
| M3 | Template ablation core | Phi-3.5, Falcon-H1-1.5B, Qwen2.5-1.5B across NQ/SQuAD P0/P3 and E2/E4 | Decide main paper table | Medium-high | llama.cpp wrapper complexity |
| M4 | MT-Bench targeted ablation | Same 3 models, P0/P3, E2 or E4 | Phi collapse and Falcon/Qwen comparison clarified | High due to judging | Judge cost/time |
| M5 | Safety extension | Category-wise table + larger validation | Safety claim can remain in main paper | Low-medium | Validation labels may expose classifier weakness |
| M6 | Optional polish | Cross-judge MT-Bench, prompt grounding expansion | Appendix support only | Variable | Can delay paper if overdone |

## Compute and Data Budget

- Total estimated GPU-hours: low for parser/safety; main cost is CPU llama.cpp inference and MT-Bench judging.
- Data preparation needs: locate raw generations; implement template wrapper; log template source per model.
- Human evaluation needs: none for must-run blocks; optional prompt-tier expansion needs annotator time.
- Biggest bottleneck: getting a faithful chat template into llama.cpp prompts without changing anything else.

## Risks and Mitigations

- Risk: W2-TEMPLATE is hard to implement consistently across Falcon-H1, Phi, and Qwen.
- Mitigation: start with Phi-3.5 because its MT-Bench collapse is the strongest diagnostic; add Falcon/Qwen after smoke success.

- Risk: Template ablation does not recover performance.
- Mitigation: paper becomes stronger by being honest: "deployment-stack boundary remains causal confound"; still keep parser sensitivity and safety improvements.

- Risk: Parser sensitivity changes absolute numbers.
- Mitigation: report parser ranges; emphasize stable qualitative conclusions.

- Risk: Safety classifier validation is weaker than expected.
- Mitigation: move safety from headline claim to cautionary diagnostic and keep category-wise transparency.

## Final Checklist

- [ ] Main paper table for E1->E2 boundary ablation is covered.
- [ ] Parser sensitivity appendix table is covered.
- [ ] Safety category-wise table and validation are covered.
- [ ] Falcon-H1 claim remains matched-scale and does not imply architecture causality.
- [ ] Nice-to-have runs are separated from must-run runs.
