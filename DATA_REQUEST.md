# PEEL — Data Request Spec (for the results repo)

Goal: produce the data that fixes the three reject-drivers from review —
**W2** (is the E2–E4 collapse quantization or lost chat template?),
**W1** (are P0–P3 tiers grounded in real user prompts?),
**W5** (no uncertainty quantification), and optionally **W3** (safety instrument).

**How to hand data back:** drop each CSV into this repo under `data/` (e.g.
`data/per_run_scores.csv`), or paste the CSV contents. Use UTF-8, comma-separated,
header row exactly as specified. Use the **corrected scorer** everywhere
(first-non-empty-line truncation + EOS/stop-token strip, same as the SQuAD/HaluEval
fix already in the pipeline). Keep the same 200 examples per run and the same fixed
seed as the main results.

---

## 0. Backbone — `per_run_scores.csv`  (FROM EXISTING RESULTS, no new runs)

One long-format row per (model, setting, task, prompt_tier). This single file lets
me recompute every main grid **and** add bootstrap confidence intervals (W5) without
re-running anything. This is the highest-priority deliverable and costs zero GPU.

| column        | type    | notes |
|---------------|---------|-------|
| `model`       | str     | canonical key, e.g. `falcon-h1-1.5b` |
| `family`      | str     | `Falcon-H1`, `Qwen`, `Gemma`, ... |
| `params_b`    | float   | billions, e.g. `1.5` |
| `setting`     | str     | `E0-REF` \| `E1-INT8` \| `E2-Q8` \| `E3-Q5` \| `E4-Q4` |
| `task`        | str     | `squad_v2` \| `nq_open` \| `truthfulqa_mc1` \| `halueval_general` \| `halueval_qa` \| `mtbench` |
| `prompt_tier` | str     | `P0` \| `P1` \| `P2` \| `P3` (MT-Bench: only `P0`,`P3`; HaluEval: `P0`) |
| `metric_name` | str     | `em` \| `f1` \| `acc` \| `mtbench_score` |
| `metric_value`| float   | EM/F1/acc in **percent 0–100**; MT-Bench on its **1–10** scale |
| `n_examples`  | int     | examples actually scored (200 for reference tasks; 80 for MT-Bench) |
| `seed`        | int     | |

If you can *also* export per-example correctness for the reference tasks
(`per_example_scores.csv`: `model,setting,task,prompt_tier,example_id,correct(0/1)`),
I can compute tighter within-cell CIs — nice-to-have, not required.

---

## 1. Experiment A — E1→E2 confound ablation  (NEW runs, cheap; fixes W2)

**Question we must answer:** at the E1→E2 boundary three things change at once
(precision path, HF→llama.cpp runtime, and chat-template applied→raw completion).
We need to isolate the **chat-template** variable, because right now a reviewer can
say "your E2–E4 collapse just measures that llama.cpp dropped the chat template,
not quantization."

**Fixed factors:** prompt tier = **P0** only; tasks = **`squad_v2`** and
**`nq_open`** (the two that collapse); 200 examples, same seed; corrected scorer.

**Models (subset, 4–5):** `falcon-h1-0.5b`, `falcon-h1-1.5b`, `qwen2.5-0.5b`,
`qwen2.5-1.5b` (add `llama-3.2-1b` for a 3rd family if easy).

**Conditions (rows) — the toggle that matters is `chat_template`:**

| `condition_label` | runtime      | precision | chat_template | status |
|-------------------|--------------|-----------|---------------|--------|
| `E0-REF`          | hf           | bf16      | applied       | already have |
| `E1-INT8`         | hf           | int8      | applied       | already have |
| `E2-Q8-CT`   **NEW** | llamacpp  | q8_0      | applied       | run this |
| `E2-Q8-raw`       | llamacpp     | q8_0      | raw           | already have (current E2) |
| `E0-raw`     **NEW** | hf        | bf16      | raw           | run this |

- **How to produce `applied` vs `raw` in llama.cpp:** use the chat-completion path
  (`llama-cpp-python` `create_chat_completion`, or server `/v1/chat/completions`,
  which applies the model's chat template) for `applied`; use the raw completion
  path (`create_completion` / `/completion`) for `raw`. GPU offload (`-ngl 99`) is
  fine and makes it faster; precision path is what matters, not CPU vs GPU.
- **`E0-raw`** = your normal HF pipeline but feed the prompt **without** applying
  `tokenizer.apply_chat_template` (raw string), full precision. This checks the
  chat-template effect is not runtime-specific.

**What this buys us (the predicted contrasts):**
- `E2-Q8-CT` vs `E2-Q8-raw` — quantization & runtime held fixed → **pure chat-template effect**.
- `E0-REF` vs `E0-raw` — precision & runtime held fixed → chat-template effect replicated on HF.
- `E1-INT8` vs `E2-Q8-CT` — both templates on → **pure quantization/runtime effect** (should be small).

If `E2-Q8-CT` recovers SQuAD/NQ EM toward E0 levels while `E2-Q8-raw` stays ~0%,
we can state plainly that the E2–E4 collapse is a serving-interface (chat-template)
effect, not quantization — which is exactly the claim currently deferred to "future work."

**Deliverable — `ablation_e1e2.csv`:**

| column            | notes |
|-------------------|-------|
| `model`           | |
| `family`          | |
| `params_b`        | |
| `condition_label` | one of the 5 above |
| `runtime`         | `hf` \| `llamacpp` |
| `precision`       | `bf16` \| `int8` \| `q8_0` |
| `chat_template`   | `applied` \| `raw` |
| `task`            | `squad_v2` \| `nq_open` |
| `prompt_tier`     | `P0` |
| `n_examples`      | 200 |
| `em`              | percent 0–100, corrected scorer |
| `f1`              | percent 0–100 (squad only; blank for nq) |
| `seed`            | |

Also please save the **raw generations** for ~5 examples per (model, condition, task)
in `ablation_samples.jsonl` (`{model,condition_label,task,example_id,prompt,raw_output,gold,em}`)
so I can sanity-check the scorer, not the model, is being measured.

---

## 2. Experiment B — P-tier grounding mini-study  (NO GPU; fixes W1)

**Question:** do P0–P3 resemble how real non-expert users write, and can annotators
reliably map real prompts onto the tiers? Right now each tier is one hand-authored
template, so P3→0% reads as "you wrote a broken prompt," not "users cause failure."

**Protocol (cheap, human annotation):**
1. Sample **~50 single-turn, English, QA-style user prompts** from a public real-user
   corpus (WildChat, LMSYS-Chat-1M, or ShareGPT — pick one; filter to single-turn
   question/task requests, strip system prompts). Record the source + filter rule.
2. Have **2–3 annotators independently** assign each prompt the best-matching tier
   **P0/P1/P2/P3**, using the operational-rule table already in the paper appendix
   (P0 expert/labelled, P1 casual-but-clear, P2 minimal/unframed, P3 informal/noisy).
3. That's it — I'll compute Fleiss'/Cohen's kappa and the tier distribution, or you
   can. The two things the paper needs: **(a)** annotators agree above chance
   (kappa), and **(b)** real prompts populate P1–P3 (i.e., real users are not all P0).

**Deliverable — `ptier_grounding.csv`:**

| column | notes |
|--------|-------|
| `prompt_id` | |
| `source` | `wildchat` \| `lmsys` \| `sharegpt` \| `manual` |
| `raw_prompt_text` | the real user prompt |
| `ann1_tier` | `P0`–`P3` |
| `ann2_tier` | `P0`–`P3` |
| `ann3_tier` | `P0`–`P3` (blank if only 2 annotators) |

Plus a 3–4 sentence note on sampling/filtering (goes into Methods + Limitations).
Optional stretch (needs GPU): for ~10 real P1/P3 prompts, run one model and check EM
is in the same ballpark as the hand-crafted template of that tier (template-sensitivity).

---

## 3. Experiment C — Safety instrument validation  (OPTIONAL; fixes W3)

Only if we want to keep safety as a headline contribution. Otherwise we demote it to
a scoped appendix diagnostic (I can do that with no new data).

**Protocol:** take a random **~120-response subset** of the existing safety generations
(mix of unsafe + safe prompts, across settings). Re-label each with a stronger judge
(GPT-4o `refuse/comply/ambiguous`) and, ideally, 1 human on ~40 of them. I'll compute
keyword-vs-LLM agreement, keyword-vs-human agreement, and recompute refusal rate +
attack-success-rate on the validated subset.

**Deliverable — `safety_validation.csv`:**

| column | notes |
|--------|-------|
| `response_id` | |
| `model` | |
| `setting` | `E0-REF`..`E4-Q4` |
| `prompt_type` | `unsafe` \| `safe` |
| `prompt_text` | |
| `model_response` | |
| `keyword_label` | `refuse` \| `comply` \| `ambiguous` |
| `llm_judge_label` | `refuse` \| `comply` \| `ambiguous` |
| `human_label` | optional, same set |

---

## Priority / effort summary

| # | Deliverable | Fixes | GPU cost | Priority |
|---|-------------|-------|----------|----------|
| 0 | `per_run_scores.csv` | W5 + lets me redraw all grids | none | **do first** |
| A | `ablation_e1e2.csv` (+ samples) | W2 (biggest credibility fix) | tiny (5 small models × 2 tasks × 2 new conditions) | **high** |
| B | `ptier_grounding.csv` | W1 (validates core premise) | none (human) | **high** |
| C | `safety_validation.csv` | W3 | tiny (judge API only) | optional |

Once #0 is back I can add CIs immediately; once A and B are back I can write the two
new subsections and upgrade the claims. Then we re-run the external cross-model review
before submission.
