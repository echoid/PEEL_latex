# Paper Plan — PEEL Benchmark

**Title**: Can Small Language Models Serve Ordinary Users? A Lower-Bound Benchmark under Non-Expert Prompts and Edge Constraints
**Benchmark name**: PEEL (Prompt–Edge Evaluation Ladder)
**One-sentence contribution**: PEEL是首个同时跨越5个边缘部署级别×4个提示质量层级的评测框架，揭示了提示质量——而非量化压缩——才是小语言模型在真实边缘场景下的主要失效驱动因素。
**Venue**: AAAI 2027
**Type**: Empirical / Diagnostic Benchmark paper
**Date**: 2026-07-06
**Page budget**: 7 technical pages (main body through Conclusion) + references (separately)
**Section count**: 7 sections

---

## 核心研究问题

> **Can small language models (≤3.8B) reliably serve ordinary, non-expert users on resource-constrained devices?**

"Lower-bound" framing: 我们不测最好情况，而是测**最差合理情况**——
非专业提示 + 最大量化压缩——作为真实部署能力下界。

---

## Claims-Evidence Matrix

| # | Claim | Evidence | Status | Section |
|---|-------|----------|--------|---------|
| C1 | 量化（E0→E4）对选择题/分类任务影响有限 | TruthfulQA: 37.9%→33.1% (−4.8pp); HaluEval-General: 57.1%→43.5% | Supported | §4 |
| C2 | 提示退化（P0→P3）是span-extraction任务的主要失效驱动 | SQuAD EM: P0均值~21% → P3 **0%**（所有模型、所有设置） | Supported | §4 |
| C3 | NQ-Open在E2–E4出现"双重崩溃"：chat template丢失 + 小模型参数知识薄弱 | NQ EM: E0~1.5% → E2–E4 **0%**（P0下） | Supported | §4 |
| C4 | Falcon-H1（SSM-Hybrid）指令遵循大幅优于同参数量Transformer | MT-Bench E0/P0: FH1-1.5B=8.20 vs Qwen-1.5B=4.06, Llama-3B=3.93 | Supported | §5 |
| C5 | Falcon-H1对提示质量退化近乎免疫，Transformer家族则显著下降 | P0→P3 Δ: FH1-1.5B=+0.075（稳定），Qwen-1.5B=−0.575，Gemma-1B=−0.54 | Supported | §5 |
| C6 | llama.cpp输出冗余导致系统性评分低估 | SQuAD: 修复EOS后 0%→21%; HaluEval: 修复首行后 24%→47–51% | Supported | §4 |
| C7 | 小模型安全拒绝率普遍极低，边缘部署存在安全隐患 | 75个model×setting对：mean refusal=6.5%, max=35.4% | Supported | §6 |
| C8 | 量化对指令遵循（MT-Bench）的影响微弱 | E0→E4 MT-Bench均值降幅 < 0.35分 | Supported | §5 |

**已知局限（投稿时需在paper中披露）：**
- phi3.5-mini / phi4-mini 从MT-Bench排除（chat template不兼容→空输出）
- InternLM2.5-1.8B 缺少E0/E1（HPC tokenizer崩溃）
- NQ-Open基线本身极低（1.3%@E0）——小模型参数知识不足，非pipeline问题
- P1–P2提示模板为手工构造，未经用户研究验证
- 仅英文数据集

---

## 各章节规划

### §0 Abstract（目标150–200词）

- **What**: PEEL，首个系统性覆盖 edge deployment × prompt quality 两个独立维度的benchmark
- **Why it matters**: 真实用户 = 非专家提示 + 量化模型；现有benchmark不捕捉此交叉
- **Setup**: 18个模型，5个边缘设置，4个提示层级，6个任务，1184+次推理
- **Key results（需精确写入）**:
  - SQuAD EM: P3下**所有模型、所有设置均归零**（prompt dominance）
  - TruthfulQA量化降幅仅4.8pp（quantization resilience）
  - Falcon-H1-1.5B MT-Bench=8.20 vs Qwen-1.5B=4.06
  - 安全拒绝率mean=**6.5%**（low refusal, 非47%——旧草稿数字有误，需纠正）
- **Punch line**: 普通用户视角下，提示质量比硬件约束更危险

**⚠️ 注意**: 旧draft里 47.3% unsafe refusal / 19.1% over-refusal 是错误数字，
实际数据：mean=6.5%, max=35.4%（来自results/safety_summary.csv）。

---

### §1 Introduction（目标1–1.5页）

- **Opening hook**: 小语言模型正在走向边缘设备，但真实用户不会写专业提示
- **Gap**: HELM/MMLU/MT-Bench均假设云端全精度+专家提示，两个维度都不符合真实场景
- **Lower-bound framing**（新增，区别于prior work）:
  > "We measure a lower bound: the worst-case realistic performance when both axes are simultaneously stressed."
- **贡献列表（必须取消注释、写入正文）**:
  1. **PEEL框架**: 5个edge设置 × 4个prompt层级的2D评测空间，定义"lower bound deployment"
  2. **Prompt分类法P0–P3**: 四层提示退化梯度，对每个数据集手工设计模板
  3. **三类任务交互规律**: MC任务双轴鲁棒、span-extraction仅对prompt敏感、知识密集在E1→E2出现cliff
  4. **评分artifact诊断**: 两个llama.cpp特有的系统性评分低估bug（EOS + first-line）及修复方法
  5. **架构洞见**: Falcon-H1（SSM-Hybrid）在边缘部署下对提示退化近乎免疫
  6. **安全发现**: 边缘小模型安全对齐脆弱（mean refusal=6.5%）
- **Hero figure**: Fig 1 — SQuAD EM 2D heatmap（5×4格）+ TruthfulQA对比
  - 视觉要点：P3列全为0（红），P0列稳定（绿）；右侧TruthfulQA无崩溃
- **Key citations**: HELM, MMLU, MT-Bench, llama.cpp/GGUF, Falcon-H1, PromptBench

---

### §2 Related Work（目标0.75–1页）

按**方法论家族**组织，不要逐篇列举：

- **LLM评测基准**: HELM, BIG-Bench, MMLU, MT-Bench — 均假设专家提示+全精度
- **量化与边缘部署**: GPTQ, GGUF, bitsandbytes, SqueezeLLM — 固定提示质量
- **提示鲁棒性**: PromptBench, RobustAlpacaEval, Sclar et al. — 不交叉部署约束
- **新型架构**: Falcon-H1, Mamba, Hymba — PEEL首次在边缘部署下量化其提示鲁棒性
- **安全评测**: HarmBench, SafetyBench — 不同时考虑量化对对齐的影响
- **定位**: 无任何先前工作同时在两个轴上变化；PEEL填补此2D gap

---

### §3 PEEL设计（目标1.5页）

- **2D空间定义**（配图：Benchmark space schematic，Fig 4）:
  - X轴：提示质量层级 P0（专家）→ P3（破碎）
  - Y轴：边缘部署层级 E0（BF16/GPU）→ E4（Q4/CPU）
- **边缘设置（表格）**:

  | 设置 | 框架 | 精度 | 硬件 | 代表配置 |
  |------|------|------|------|---------|
  | E0-REF | HF Transformers | BF16 | GPU | A100 |
  | E1-INT8 | bitsandbytes | INT8 | GPU | A100 |
  | E2-Q8 | llama.cpp | Q8_0 | CPU | 消费级 |
  | E3-Q5 | llama.cpp | Q5_K_M | CPU | 消费级 |
  | E4-Q4 | llama.cpp | Q4_K_M | CPU | 超低资源 |

- **提示层级（表格 + SQuAD示例）**:

  | 层级 | 名称 | 描述 | 示例特征 |
  |------|------|------|---------|
  | P0 | Expert | 完整指令+格式说明 | JSON指定，角色提示 |
  | P1 | Casual | 自然语言，无结构化指令 | 口语化，缺乏格式要求 |
  | P2 | Naive | 极简、无上下文 | 仅问题，无任何引导 |
  | P3 | Broken | 语法错误+模糊描述 | 拼写错误，语序混乱 |

- **数据集**: TruthfulQA-MC1, SQuAD v2, NQ-Open, HaluEval (P0 only), MT-Bench (P0/P3), Safety (HarmBench)
- **模型表格（Table 1）**: 18个模型，9个家族，0.09B–3.8B，E0–E4覆盖情况
- **实验协议**: 200 examples/run（MT-Bench: 80），固定seed，离线评分，SLURM HPC

---

### §4 Results: 量化×提示交互（目标2–2.5页）

#### §4.1 TruthfulQA-MC1: 双轴鲁棒
- Mean Acc: 37.9%（E0）→ 33.1%（E4），Δ=−4.8pp
- P0→P3降幅 <5pp
- **所需表格**: Table 2 — 5(setting)×4(tier) EM均值网格
- **结论**: MC格式同时吸收量化噪声和提示噪声

#### §4.2 SQuAD v2: 提示主导型崩溃
- P0 EM: 19.8–22.0%（E0–E4稳定）
- P3 EM: **0.0%（所有模型，所有设置）**
- **所需图表**: Fig 1（Hero）— 左: absolute heatmap (YlGnBu)，右: Δ vs E0-REF (RdBu)
- **所需表格**: Table中给出P0/P3代表数字
- **Artifact披露**: 初始E2+ EM=0%因EOS/stop-token评分bug；修复后恢复21%

#### §4.3 NQ-Open: E2–E4双重崩溃
- E0/E1: 1.3–1.7% EM（本身极低——小模型参数知识薄弱）
- E2–E4: **0.0% EM**
- 双重原因：(1) llama.cpp丢失chat template → 模型未进入指令遵循模式；(2) 小模型参数知识不足
- **所需图表**: Fig（NQ heatmap，可复用heatmap格式）或Table 4

#### §4.4 HaluEval: Artifact + 修复
- General: 57.1%→43.5%（E0→E4）；QA: 41.1%→43.9%
- Artifact: 首行解析bug导致50–62% E2–E4结果标记为"ambiguous"；首行截断后恢复47–51%
- **与SQuAD artifact同根因**: llama.cpp verbose output的系统性影响

---

### §5 Architecture Sensitivity: Instruction-Following（目标1页）

#### §5.1 家族比较（E0/P0基准）
- **所需表格**: Table 3 — MT-Bench by family × setting × tier

  | Family | E0/P0 | E4/P0 | P0→P3 Δ@E0 |
  |--------|-------|-------|------------|
  | Falcon-H1 | 6.84 (avg) / **8.20** (1.5B) | — | **+0.075** |
  | Gemma | 7.43 | — | −0.54 |
  | Llama | ~5.5 | — | varies |
  | Qwen | 3.45 (avg) / 4.06 (1.5B) | — | −0.575 |

- 关键数字: FH1-1.5B=8.20 vs Qwen-1.5B=4.06（gap=4.14 points）
- **所需图表**: Fig 3 — per-model bar chart（E0/P0 solid，P3 transparent），按family着色

#### §5.2 量化对指令遵循的影响
- E0→E4 MT-Bench降幅 <0.35分（量化鲁棒）
- Setting均值: E0=4.93, E1=4.99, E2=4.97, E3=4.69, E4=4.59
- **结论**: 指令遵循对量化鲁棒，但对架构选择极为敏感

#### §5.3 提示鲁棒性与架构的关联
- Falcon-H1（SSM-Hybrid hybrid-attention）提供结构性提示鲁棒性
- Dense Transformer在P3下显著退化
- **所需图表**: 可用Fig 3中P3透明bar直观呈现；或单独雷达图（MT-Bench by category at P0 vs P3）

---

### §6 Safety Evaluation（目标0.5页）

- Setup: HarmBench（不安全提示）× 16模型 × up to 5个设置
- **关键数字（已从data验证）**:
  - Mean refusal rate: **6.5%**（75个model×setting对）
  - Max: **35.4%**（单个模型最高）
  - 多个模型接近**0%**拒绝率
- **所需图表**: Fig 5 — Safety refusal rate heatmap（model × setting，Oranges colormap）
- **边缘效应**: 量化对拒绝率的影响方向不一致（无规律性增减）
- **结论**: 亚2B规模模型安全对齐普遍脆弱；边缘量化不会系统性改善或恶化安全行为

---

### §7 Discussion + Conclusion（目标0.5页）

- **统一规律**: 任务类型调节两个轴的影响
  - MC/分类：双轴鲁棒
  - Span-extraction：仅prompt敏感
  - 知识密集：E1→E2 cliff
  - 指令遵循：架构敏感，量化鲁棒
- **Artifact教训**: 任何llama.cpp评测若不做首行截断，系统性低估span/分类任务
- **实践建议**:
  - 边缘量化部署优先选MC/分类任务，避免span-extraction
  - 提示鲁棒性关键场景优选Falcon-H1
  - 边缘部署前必须独立测试安全对齐
- **局限性（必须在正文披露）**: 英文only; P1–P3手工构造; phi/InternLM覆盖缺失
- **Future work**: 多语言; P1–P3用户研究验证; 大规模安全红队测试; E1→E2 GPU-GGUF ablation

---

## 现有图片资产清单（results/figures/）

> 所有图均为 PDF 格式，可直接插入 LaTeX。脚本全部可复现。
> 重新生成：`python analysis/gen_figures.py`（主文图）+ `python analysis/gen_appendix_figs.py`（附录图）

### 正文图（Main Paper Figures）— 5张，已生成 ✅

| 文件名 | 用途 | 建议放置 |
|--------|------|---------|
| `fig1_squad_heatmap.pdf` | SQuAD v2 Hero：2D heatmap（absolute + Δ vs E0），展示P3全崩溃 | §4.2，主文 Fig 1 |
| `fig2_tqa_heatmap.pdf` | TruthfulQA：同格式，与 Fig1 对比展示"无崩溃" | §4.1，主文 Fig 2 |
| `fig3_mtbench_family_bar.pdf` | MT-Bench per-model bar，P0实心/P3透明，按family着色 | §5，主文 Fig 3 |
| `fig4_nq_heatmap.pdf` | NQ-Open：展示 E2–E4 双重崩溃 | §4.3，主文 Fig 4 |
| `fig5_safety_heatmap.pdf` | Safety 拒绝率 heatmap（model × setting） | §6，主文 Fig 5 |

**还缺一张**：Fig 4（PEEL benchmark space 示意图，5×4格+坐标轴）需手工绘制（Keynote / Inkscape / TikZ）。

---

### 附录图（Appendix Figures）— 29张，已生成 ✅

#### A. 各数据集详细分析（每个数据集4张，共20张）

| 数据集 | 文件（前缀） | by_prompt | by_edge | heatmap | by_family |
|--------|-------------|-----------|---------|---------|-----------|
| SQuAD v2 | `app_squad_v2_` | ✅ | ✅ | ✅ | ✅ |
| NQ-Open | `app_nq_open_` | ✅ | ✅ | ✅ | ✅ |
| TruthfulQA-MC1 | `app_truthfulqa_mc1_` | ✅ | ✅ | ✅ | ✅ |
| HaluEval-General | `app_halueval_general_` | ✅ | ✅ | ✅ | ✅ |
| HaluEval-QA | `app_halueval_qa_` | ✅ | ✅ | ✅ | ✅ |

- `_by_prompt.pdf`：各E-setting线图，x轴=P-tier；**左图带std阴影，右图不带**
- `_by_edge.pdf`：各P-tier线图，x轴=E-setting；**左图带std阴影，右图不带**
- `_heatmap.pdf`：每个模型 × P-tier 的全量 heatmap（按family排序）
- `_by_family.pdf`：按family分组的bar chart，误差棒=family内std

#### B. MT-Bench 详细分析（5张）

| 文件名 | 描述 |
|--------|------|
| `app_mtbench_by_edge.pdf` | 各family跨E-setting均值线图（P0和P3各一子图） |
| `app_mtbench_by_family_prompt.pdf` | P0 vs P3 family对比bar（E0-REF，带std） |
| `app_mtbench_radar_P0.pdf` | 8类别雷达图：所有E-setting在P0 |
| `app_mtbench_radar_P3.pdf` | 8类别雷达图：所有E-setting在P3 |
| `app_mtbench_radar_E0E4.pdf` | E0 vs E4叠加，P0实线/P3虚线 |

#### C. Safety 详细分析（4张）

| 文件名 | 描述 |
|--------|------|
| `app_safety_refusal.pdf` | 拒绝率 heatmap（HarmBench不安全提示，Oranges） |
| `app_safety_fpr.pdf` | 过度拒绝率 heatmap（XSTest安全提示，Purples） |
| `app_safety_by_category.pdf` | 按危害类别分组的拒绝率bar（6个类别 × 各family） |
| `app_safety_scatter.pdf` | 拒绝率 vs 过度拒绝率散点图（安全-可用性 tradeoff） |

---

### 旧版/历史图（results/figures/ 中存在但不再使用）

以下是从旧 presentation/ 和早期 notebook 版本留下的图，**不要插入论文**，仅作参考：

`fig_arch.pdf`, `fig_squad.pdf`, `fig_mtbench_edge_settings.pdf`, `fig_mtbench_p0p3_per_model.pdf`, `fig_mtbench_p0p3_shrinkage.pdf`, `fig_mtbench_per_model.pdf`, `fig_mtbench_settings_per_model.pdf`, `fig_mtbench_radar.pdf`, `fig1_benchmark_space.pdf`, `fig1_heatmap.pdf`, `fig2_curves.pdf`, `fig2_main_heatmap.pdf`, `fig3_arch.pdf`, `fig3_degradation_curves.pdf`, `fig4_arch_comparison.pdf`, `fig4_squad_tiers.pdf`, `fig5_squad_collapse.pdf`, `mtbench_radar.svg`

---

## 可生成的表格清单

> 以下表格**不需要额外跑实验**，直接从现有数据生成 LaTeX 代码即可。

### 主文表格

| ID | 描述 | 数据来源 | 生成方式 |
|----|------|---------|---------|
| **Table 1** | 模型表：18模型 × family × 参数量 × E0–E4覆盖情况 | 手工整理 | 写入 LaTeX tabular |
| **Table 2** | TruthfulQA 5(setting)×4(tier) Acc% 均值网格（含行列均值） | `results/summary.csv` | `analysis/aggregate_results.py` |
| **Table 3** | MT-Bench：family × {P0,P3} 得分，含Δ列 | `results/summary.csv` | 直接从数据计算 |
| **Table 4** | SQuAD v2 5×4 EM% 网格（主文简化版或附录全版） | `results/summary.csv` | `analysis/aggregate_results.py` |
| **Table 5** | 提示层级定义表：P0–P3描述+SQuAD示例 | 手工 | 写入 LaTeX tabular |
| **Table 6** | 边缘设置定义表：E0–E4框架/精度/硬件 | 手工 | 写入 LaTeX tabular |

### 附录表格

| ID | 描述 | 数据来源 |
|----|------|---------|
| App Table A | NQ-Open 5×4 EM% 网格（展示E2–E4=0%） | `results/summary.csv` |
| App Table B | HaluEval-General 5×1 Acc%（P0 only，E-setting变化） | `results/summary.csv` |
| App Table C | HaluEval-QA 同上 | `results/summary.csv` |
| App Table D | Safety 全表：16模型 × 5设置 × refusal/FPR | `results/safety_summary.csv` |
| App Table E | Safety 按类别：6类危害的拒绝率（家族均值） | `results/safety_summary.csv` |
| App Table F | MT-Bench 按category × setting（8类×5设置） | `results/mtbench_by_category.csv` |
| App Table G | Failure case taxonomy（手工分析典型错误） | 手工 |

---

## 图表计划（论文正文布局建议）

### 正文图（7页预算内）

| ID | 类型 | 描述 | 数据来源 | 优先级 |
|----|------|------|---------|--------|
| **Fig 1** | **2D Heatmap pair（Hero）** | **SQuAD EM absolute + Δ vs E0，P3列全红** | `fig1_squad_heatmap.pdf` ✅ | **必须** |
| **Fig 2** | 2D Heatmap pair | TruthfulQA，与Fig1对比无崩溃 | `fig2_tqa_heatmap.pdf` ✅ | **必须** |
| **Fig 3** | Bar chart | MT-Bench per-model P0 vs P3，family着色 | `fig3_mtbench_family_bar.pdf` ✅ | **必须** |
| **Fig 4** | 示意图 | PEEL benchmark space 5×4格子+轴标注 | ❌ 需手工绘制 | 高 |
| **Fig 5** | Safety heatmap | 拒绝率 model × setting | `fig5_safety_heatmap.pdf` ✅ | 高 |

**Fig 1 Caption草案**：
> "PEEL reveals that prompt quality, not quantization, drives SQuAD v2 failure. EM collapses to 0% at P3 universally, regardless of edge setting (rows), while E0→E4 quantization has negligible effect at P0 (left column)."

### 正文表格（7页预算内）

| ID | 描述 | 优先级 |
|----|------|--------|
| **Table 1** | 模型表 | **必须** |
| **Table 2** | TruthfulQA 5×4网格 | **必须** |
| **Table 3** | MT-Bench family对比 | **必须** |
| Table 4 | 提示层级定义 | 高（§3） |
| Table 5 | 边缘设置定义 | 高（§3） |

---

## 命名一致性问题（⚠️ 投稿前必须解决）

- **Abstract**: 已使用"PEEL"
- **Body text**: 仍使用"EdgeBench"
- **建议**: 统一用"PEEL"，在§3开头定义全称"Prompt–Edge Evaluation Ladder (PEEL)"
- **动作**: 搜索所有"EdgeBench"替换为"PEEL"

---

## 错误数字清单（⚠️ 必须在写作前修正）

| 位置 | 错误数字 | 正确数字（来自data） |
|------|---------|-------------------|
| Abstract安全描述 | "47.3% unsafe refusal / 19.1% over-refusal" | mean refusal=**6.5%**, max=**35.4%** |
| Abstract run count | "1,147 runs" | **1,184+** filtered runs |
| MT-Bench Δ | "SSM-Hybrid Δ=−0.05" | FH1-1.5B P0→P3 = **+0.075**（改善不是退化！） |

---

## Citation计划

- §1 Intro: HELM, MMLU, MT-Bench, GPTQ, GGUF/llama.cpp, Falcon-H1, PromptBench
- §2 Related: BIG-Bench, SuperGLUE, RobustAlpacaEval, SqueezeLLM, Mamba, Hymba, HarmBench
- §3 Design: SQuAD v2, NQ-Open, TruthfulQA, HaluEval, bitsandbytes
- §5 Architecture: Falcon-H1, Mamba, Hymba
- §6 Safety: HarmBench, GPT-4o-mini judge

**⚠️ Citation验证**: 投稿前所有引用需与references.bib核对。HarmBench, bitsandbytes, PalmBench需[VERIFY]。

---

## 写作优先顺序（AAAI 7页预算）

1. **Fig 1 + Table 2**: Hero图 + TruthfulQA表格（最先完成，定基调）
2. **§4 Results**: 5个子节填入真实数字，无任何估算
3. **§3 PEEL Design**: 模型表(Table 1) + 提示层级表 + 协议
4. **§1 Introduction**: 取消注释contributions列表，写入6条贡献
5. **§0 Abstract**: 用最终数字重写，修正安全数字
6. **§5 MT-Bench**: Table 3 + Fig 3，注明phi排除原因
7. **§6 Safety**: Fig 5 + 修正后安全数字
8. **§7 Discussion**: 合并Limitations + Conclusion

---

## Next Steps

- [ ] 运行 `python analysis/gen_figures.py` 重新生成 Fig1–5 (PDF → results/figures/)
- [ ] 运行 `python analysis/mtbench_radar.py --plot` 生成雷达图 (→ results/figures/)
- [ ] 新建 paper LaTeX repo，以AAAI 2027 template为基础
- [ ] `/paper-write §3` — PEEL Design section（模型表 + 协议 + 提示层级表）
- [ ] `/paper-write §4` — Results（5个子节，全用真实数字）
- [ ] `/paper-write §1` — Introduction（恢复contributions列表，lower-bound framing）
- [ ] `/paper-write §0` — Abstract重写（修正安全数字、run count、MT-Bench Δ）
- [ ] `/paper-claim-audit` — 验证每个claim都有对应图/表引用
- [ ] `/citation-audit` — 验证所有\\citep key在bib文件里存在
