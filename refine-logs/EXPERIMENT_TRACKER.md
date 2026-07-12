# Experiment Tracker

| Run ID | Milestone | Purpose | System / Variant | Split | Metrics | Priority | Status | Notes |
|--------|-----------|---------|------------------|-------|---------|----------|--------|-------|
| R001 | M0 | Reproduce current corrected metrics from raw outputs | Existing parser | SQuAD v2 + HaluEval, all current runs | EM/F1, acc, ambiguous rate | MUST | TODO | Sanity before parser sensitivity |
| R002 | M1 | Parser sensitivity A/B/C/D | Existing raw outputs | SQuAD v2, all tiers/settings | EM/F1, max delta | MUST | TODO | No new inference |
| R003 | M1 | Parser sensitivity A/B/C/D | Existing raw outputs | HaluEval, all tiers/settings | acc, ambiguous rate, max delta | MUST | TODO | No new inference |
| R004 | M2 | Template ablation smoke | Phi-3.5-Mini E2-Q8 current vs W2-TEMPLATE | SQuAD v2 P0/P3, 200 examples | EM/F1, echo/stop rate | MUST | TODO | Start here |
| R005 | M2 | Template ablation smoke | Phi-3.5-Mini E2-Q8 current vs W2-TEMPLATE | NQ-Open P0/P3, 200 examples | EM/F1, output diagnostics | MUST | TODO | Strongest runtime test |
| R006 | M3 | Core template ablation | Falcon-H1-1.5B E2-Q8/E4-Q4 W2-TEMPLATE | SQuAD + NQ P0/P3 | EM/F1, diagnostics | MUST | TODO | Primary family |
| R007 | M3 | Core template ablation | Qwen2.5-1.5B E2-Q8/E4-Q4 W2-TEMPLATE | SQuAD + NQ P0/P3 | EM/F1, diagnostics | MUST | TODO | Matched-scale baseline |
| R008 | M3 | Core template ablation | Phi-3.5-Mini E2-Q8/E4-Q4 W2-TEMPLATE | SQuAD + NQ P0/P3 | EM/F1, diagnostics | MUST | TODO | Dense Transformer collapse case |
| R009 | M4 | MT-Bench template ablation | Phi-3.5-Mini E2-Q8/E4-Q4 W2-TEMPLATE | MT-Bench P0/P3 | GPT-4o-mini score | SHOULD | TODO | Run if judge budget ok |
| R010 | M4 | MT-Bench template ablation | Falcon-H1-1.5B + Qwen2.5-1.5B W2-TEMPLATE | MT-Bench P0/P3 | GPT-4o-mini score | SHOULD | TODO | Supports matched-scale story |
| R011 | M5 | Safety category table | Existing safety outputs | Unsafe + safe prompts | refusal/compliance/ambiguous by category | SHOULD | TODO | Likely cheap |
| R012 | M5 | Safety validation extension | Stratified validation sample | 100-200 outputs | agreement, kappa, precision/recall | SHOULD | TODO | Use GPT-4o or human if possible |
| R013 | M6 | MT-Bench cross-judge spot check | Sampled 20-40 MT-Bench outputs | Key comparisons | judge correlation, rank stability | NICE | TODO | Appendix only |
| R014 | M6 | Prompt-tier grounding expansion | Extra WildChat prompts | n=100-150 total | tier distribution, kappa | NICE | TODO | Do only after must-runs |
