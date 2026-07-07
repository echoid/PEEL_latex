# PEEL — Paper-to-Data Claim Audit

Source of truth: `per_run_scores.csv` (labelled by author as "所有主实验数据"),
`safety_summary.csv`. Aggregation: mean over models per (setting, prompt_tier)
cell, reference tasks filtered to n_examples >= 100, matching the paper's stated
protocol. Date: 2026-07-07.

## Verdict
- **MT-Bench and Safety: MATCH the paper.** ✓
- **SQuAD v2, NQ-Open: DO NOT MATCH — the "collapse to 0%" hero claims are false
  against the authoritative data.** ✗ (critical)
- **TruthfulQA, HaluEval: only the E0-REF row is stale;** E1–E4 match. ✗ (moderate)
- Main reference-task tables (`tab:squad_grid`, `tab:nq_grid`, `tab:tqa_grid`,
  `tab:halueval_grid`) and Figures 1/`fig2_tqa`/`fig4_nq` must be regenerated
  from this CSV.

## SQuAD v2 EM (mean over models, from CSV)
| setting | P0 | P1 | P2 | P3 |
|---|---|---|---|---|
| E0-REF | 30.5 | 16.8 | 5.6 | 4.7 |
| E1-INT8 | 30.4 | 16.6 | 5.6 | 4.7 |
| E2-Q8 | 30.5 | 21.1 | 3.4 | 2.4 |
| E3-Q5 | 29.7 | 19.4 | 3.1 | 2.2 |
| E4-Q4 | 27.8 | 18.8 | 3.0 | 2.3 |

Paper currently claims P0≈20–22 and **P3 = 0.0 universal**. Reality: P0≈28–30,
P3≈2–5, and only **2 of 80** model×setting P3 cells are exactly 0 (both
`falcon_h1_tiny_r_06b` at E0/E1). Correct narrative: P3 causes a **~6–15× drop**
(near-total collapse), not an exact zero.

## NQ-Open EM (mean over models, from CSV)
| setting | P0 | P1 | P2 | P3 |
|---|---|---|---|---|
| E0-REF | 8.2 | 3.2 | 3.0 | 3.1 |
| E1-INT8 | 8.4 | 3.1 | 3.0 | 3.0 |
| E2-Q8 | 3.6 | 1.9 | 1.8 | 1.7 |
| E3-Q5 | 3.5 | 1.8 | 1.8 | 1.7 |
| E4-Q4 | 3.3 | 1.7 | 1.7 | 1.6 |

Paper claims E0 P0=1.3 and E2–E4 = 0.0. Reality: E0 P0=8.2 (phi4_mini 17.7,
phi35_mini 17.4, llama32_3b 12.3, falcon_h1_15b_deep 12.0), and E2–E4 P0 ≈ 3.3–3.6
(never 0). Correct narrative: quantization/runtime roughly **halves** NQ recall at
the E1→E2 boundary; it does not zero it. This is more credible than the old "0%".

## TruthfulQA-MC1 acc (from CSV) — only E0-REF row changed
| setting | P0 | P1 | P2 | P3 |
|---|---|---|---|---|
| E0-REF | 39.7 | 40.6 | 36.6 | 37.6 |  (paper: 37.9/40.4/36.7/36.6)
| E1-INT8 | 39.4 | 41.0 | 35.7 | 37.3 |  (matches)
| E2-Q8 | 40.1 | 29.1 | 30.1 | 28.8 |  (matches)
| E3-Q5 | 38.4 | 31.6 | 33.0 | 30.1 |  (matches)
| E4-Q4 | 38.6 | 30.2 | 32.9 | 30.8 |  (matches)

Row means: E0=38.6 … E4=33.1 → all-tier drop = **5.5pp** (paper says 4.8pp).
P0-only E0→E4 change = **-1.1pp** (paper says -0.7pp). Story (robust to both axes)
holds; numbers need updating.

## HaluEval acc, P0 (from CSV) — only E0-REF stale
- General: 52.2 / 51.9 / 50.8 / 46.5 / 43.5  (paper E0=57.1)
- QA:      38.2 / 39.6 / 47.3 / 47.2 / 43.9  (paper E0=41.1)

## Safety (from safety_summary.csv, 75 model×setting pairs) — MATCHES + new data
- Mean refusal 0.065 (**6.5%**) ✓, max 0.354 (**35.4%**, `falcon_h1_tiny_r_06b`@E3-Q5) ✓
- 45 / 75 pairs refuse < 5% of unsafe prompts
- **Over-refusal (FPR): mean 3.5%, max 32.8%** ← use this to replace the old wrong 19.1%

## MT-Bench — MATCHES
- Setting means E0=5.0 … E4=4.6 ✓; family: Gemma 7.40, InternLM 6.78, Falcon-H1 6.45,
  Qwen 3.61, SmolLM 3.09 ✓; FH1-1.5B=8.20, Qwen2.5-1.5B=4.06 ✓
- (Note: paper lumps "Llama/TinyLlama 3.73"; CSV separates Llama 4.32 / TinyLlama 2.55.)

## Required actions
1. Regenerate `tab:squad_grid`, `tab:nq_grid`, `tab:tqa_grid`, `tab:halueval_grid`
   from this CSV (I can produce the LaTeX directly).
2. Rewrite SQuAD narrative: "collapses to 0% universally" → "near-total collapse,
   P0≈30% → P3≈2–5% (a 6–15× drop)". Update abstract/intro/conclusion accordingly.
3. Rewrite NQ narrative: drop the "0% at E2–E4" and "~1% parametric recall" framing;
   use "recall roughly halves at the GPU→CPU boundary (8%→3.3%)".
4. Fix TruthfulQA E0 row + the 4.8pp→5.5pp / -0.7→-1.1pp figures.
5. Fix HaluEval E0 values.
6. Replace safety over-refusal 19.1% with FPR mean 3.5% / max 32.8%.
7. **Regenerate figures** `fig1_squad_heatmap`, `fig2_tqa_heatmap`, `fig4_nq_heatmap`
   in the results repo from this CSV (I only have the stale PDFs here).
8. Fix "18 models" vs 16 reference-scored models (Falcon-E excluded from reference tasks).
