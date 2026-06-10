# FrugalBench 升级版评分架构设计：融合竞品精华的完整技术方案

## 设计动机：原版 WCP 的三个不足

原版 FrugalBench 的核心指标 WCP（Weighted Cost per Point）定义为：

\[
\text{WCP} = \frac{\text{Total Real Cost}}{\sum_i w_i \cdot s_i}
\]

这一设计有三个已被竞品研究所揭示的隐患：

1. **无参照系**：WCP 是绝对货币数字，跨时间不可比（API 价格每年下降 10×），跨任务集不可比。Cost-of-Pass 通过引入"frontier cost-of-pass"（最优可达下界）解决了这个问题。[^1]
2. **无形状信息**：WCP 是单个标量，无法揭示"精度越高是否值得付出更多成本"的曲线结构。HAL 和 RouterBench 用 Pareto 前沿和 AIQ 积分指标补充了这一维度。[^2][^3]
3. **无 Gaming 防御**：系统可以刷大量低难度题来拉低 WCP，而实际上回避所有高难度任务。CostBench 的路径偏差（Cost Gap）和 RouterBench 的凸包结构为此提供了对抗思路。[^4][^5]

以下提出吸收三大竞品精华的升级版评分架构，命名为 **FrugalScore v2**。

***

## 一、FrugalScore v2 的整体架构

FrugalScore v2 采用**三层评分体系**：

```
Layer 1: nWCP   — 归一化成本效率主指标（核心排名依据）
Layer 2: CEI    — 成本效率改进指数（相对前沿的距离）
Layer 3: Profile — 五维雷达图（可解释性附加维度）
```

主指标只有一个（nWCP），它是可排名的单一标量，方便学术比较。CEI 提供相对参照，回答"比最优基线好多少"。Profile 雷达图提供细粒度可解释性，帮助研究者诊断系统弱点。

***

## 二、Layer 1：nWCP — 归一化加权成本效率

### 2.1 原始 WCP 的归一化

**灵感来源**：Cost-of-Pass 的 frontier cost-of-pass，RouterBench 的 Zero Router 基线。[^3][^1]

为解决 WCP 绝对值跨时间不可比的问题，引入**任务难度基准成本** \(C^*_i\)，定义为在第 \(i\) 个任务上、使用当前定价下 cheap model only 基线的实测成本：

\[
C^*_{\text{total}} = \sum_{i=1}^{N} C^*_i
\]

其中 \(C^*_i\) 是 cheap baseline 在任务 \(i\) 上的实际花费（含 API + 工具 + 执行费用）。

**归一化 WCP（nWCP）** 定义为：

\[
\text{nWCP} = \frac{\text{WCP}_{\text{system}}}{\text{WCP}_{\text{cheap\_baseline}}} = \frac{C_{\text{total}} / \sum_i w_i s_i}{C^*_{\text{total}} / \sum_i w_i s^*_i}
\]

其中 \(s^*_i\) 是 cheap baseline 在任务 \(i\) 的得分，\(s_i\) 是被测系统的得分。

**解读**：
- nWCP = 1.0：与 cheap baseline 同等成本效率
- nWCP < 1.0：**优于** cheap baseline（越低越好）
- nWCP > 1.0：成本效率不如 cheap baseline

这一归一化使指标**无量纲**，不受 API 定价变化影响，任何时间的实验结果均可横向对比。[^6][^1]

### 2.2 难度权重的更新：自适应 Pass-Rate 权重

**灵感来源**：CostBench 的任务内在复杂度设计，Cost-of-Pass 对任务经济价值的建模。[^1][^4]

原版 FrugalBench 使用固定的 1/2/4/8 权重，但这一设计可被攻击：若高难度任务对应的 API 成本本身也高，则难度权重无法真正激励系统解决硬题。

改进方案：难度权重由**模型群 pass rate** 自动推导：

\[
w_i = \frac{1}{p_i + \epsilon}
\]

其中 \(p_i\) 是"所有 baseline 系统在任务 \(i\) 上的平均通过率"，\(\epsilon = 0.01\) 为平滑项。这样：
- 所有基线都能解的任务：\(p_i \approx 1.0\)，\(w_i \approx 1\)（低权重）
- 没有任何基线能解的任务：\(p_i \approx 0.0\)，\(w_i\) 趋于最大（高权重）

这与 Cost-of-Pass 中"正确解的期望价值与其稀缺性正相关"的思想一致，同时避免了预设权重被提前 overfit 的问题。[^7]

**离散化**（可选，便于工程实现）：

\[
w_i = \begin{cases} 1 & \text{if } p_i > 0.75 \text{ (Easy)} \\ 2 & \text{if } 0.40 < p_i \le 0.75 \text{ (Medium)} \\ 4 & \text{if } 0.15 < p_i \le 0.40 \text{ (Hard)} \\ 8 & \text{if } p_i \le 0.15 \text{ (Expert)} \end{cases}
\]

***

## 三、Layer 2：CEI — 成本效率改进指数

**灵感来源**：RouterBench 的 AIQ 积分指标，HAL 的 Pareto frontier，"AI Agents That Matter" 的联合优化框架。[^8][^9][^2][^3]

nWCP 是单点静态指标，无法反映系统在不同"成本预算"约束下的鲁棒性。CEI 通过对成本-性能曲线的积分来解决这个问题。

### 3.1 构建非递减凸包（Pareto Frontier）

对任意系统 \(S\)，通过调整其内部参数（如路由阈值、反思次数、agent 数量等），可以得到一系列 (成本, 加权得分) 数据点：

\[
\{(C_{\theta_1}, Q_{\theta_1}), (C_{\theta_2}, Q_{\theta_2}), \ldots, (C_{\theta_k}, Q_{\theta_k})\}
\]

参照 RouterBench 的方法，对这些点构建**非递减凸包** \(\widetilde{S}\)：确保在相同成本下取最高得分，在相同得分下取最低成本。[^3]

### 3.2 CEI 定义

定义"理论上界曲线" \(\text{Oracle}(c)\) 为每个成本水平 \(c\) 上所有系统和基线的最大可达得分（即 Oracle Router），以及"下界" \(\text{Cheap}(c)\) 为 cheap baseline 曲线：

\[
\text{CEI}(S) = \frac{\int_{c_{\min}}^{c_{\max}} \bigl[\widetilde{S}(c) - \text{Cheap}(c)\bigr]\, dc}{\int_{c_{\min}}^{c_{\max}} \bigl[\text{Oracle}(c) - \text{Cheap}(c)\bigr]\, dc}
\]

**解读**：
- CEI = 0：系统效率等同于 cheap baseline
- CEI = 1：系统达到 Oracle Router 上界
- CEI < 0：系统在所有成本预算下均劣于 cheap baseline（表示 Multi-Agent 开销完全无意义）

CEI 的量纲为"相对于最优空间的面积占比"，完全无量纲，跨数据集可比。[^2][^3]

***

## 四、Layer 3：五维可解释性 Profile

**灵感来源**：HAL 的多维度分析（精度、成本、支架设计），CostBench 的 Cost Gap / Edit Distance 多指标，"AI Agents That Matter" 对 benchmark gaming 的讨论。[^10][^11][^4][^8]

单一指标（nWCP / CEI）无法帮助研究者定位系统的具体弱点。五维 Profile 将系统的表现分解为五个正交维度：

### Dimension 1 — P（Performance）：加权精度

\[
P = \frac{\sum_i w_i s_i}{\sum_i w_i}
\]

归一化到 \([0,1]\)，反映系统在难度加权任务上的平均精度。**越高越好**。

### Dimension 2 — E（Efficiency）：成本效率

\[
E = 1 - \text{nWCP} \text{ (clipped to } [0, 1]\text{)}
\]

即 nWCP 的镜像，统一转化为"越高越好"的方向。当 nWCP > 2 时截断为 0。

### Dimension 3 — O（Overhead Penalty）：协调成本惩罚

**灵感来源**：FrugalBench 原始设计，HAL 发现 coordination overhead 是主要成本来源。[^11]

\[
O = 1 - \frac{C_{\text{coord}}}{C_{\text{total}}}
\]

其中 \(C_{\text{coord}}\) 是系统总成本中来自 planner call、reflection、debate、verification 等协调步骤的部分。\(O = 1\) 表示零协调开销（最高效），\(O \to 0\) 表示协调成本占据全部花费（"忙而无益"系统）。

这一维度直接回答 HAL 提出的核心问题："更高的推理强度在多数情况下实际降低了精度"。[^11]

### Dimension 4 — R（Robustness）：难度分布鲁棒性

\[
R = 1 - \frac{\sigma(\{r_d\}_{d \in \{\text{Easy, Medium, Hard, Expert}\}})}{0.5}
\]

其中 \(r_d\) 是系统在难度层级 \(d\) 上的"单位成本得分"（类 nWCP，按难度分层计算），\(\sigma\) 为标准差，除以 0.5 做最大值归一化。

**解读**：若系统在简单任务上成本极低但在困难任务上成本爆炸，\(R\) 值低。鲁棒的系统应在各难度下保持相近的成本效率。这从根本上杜绝"刷简单题"的 Gaming 行为。[^8]

### Dimension 5 — S（Stability）：重复运行稳定性

\[
S = 1 - \frac{\text{CV}(\text{WCP across repeated runs})}{1.0}
\]

其中 \(\text{CV}\) 为变异系数（标准差/均值），上限截断到 1。HAL 在其大规模实验中发现，LLM 调用的随机性导致成本方差可达均值的 30%–50%，不纳入稳定性指标会严重误导结论。建议每个系统至少运行 3 次取均值和 CV。[^2][^11]

### 五维雷达图示意

五个维度 P / E / O / R / S 均归一化至 \([0,1]\)，且均为"越高越好"，可以在同一雷达图上直观比较不同系统的优劣势模式。例如：
- **Cheap-Only**：P 低，E 高，O 高（无协调），R 中等，S 高 → 五边形矮但均匀
- **Multi-Agent Debate**：P 高，E 低，O 低（大量协调），R 可能好，S 中等 → 形状偏斜
- **Weak-Strong Router**：P 中高，E 中高，O 中等，R 高，S 中 → 理想的均衡五边形

***

## 五、完整评分公式汇总

### 主排行榜公式

\[
\boxed{\text{nWCP} = \frac{C_{\text{sys}} / \sum_i w_i s_i}{C^*_{\text{cheap}} / \sum_i w_i s^*_i}}
\]

次级排名（可选）：\(\text{CEI}\)（积分型），用于"在给定成本预算下哪个系统最优"类问题。

### 五维 Profile 汇总

| 维度 | 符号 | 公式 | 方向 | 吸收自 |
|------|------|------|------|--------|
| 加权精度 | P | \(\sum w_i s_i / \sum w_i\) | ↑ | FrugalBench 原始 |
| 成本效率 | E | \(1 - \text{nWCP}\) | ↑ | Cost-of-Pass[^1] |
| 协调惩罚 | O | \(1 - C_{\text{coord}}/C_{\text{total}}\) | ↑ | HAL[^11] |
| 难度鲁棒 | R | \(1 - \sigma(\{r_d\})/0.5\) | ↑ | RouterBench AIQ[^3] |
| 稳定性 | S | \(1 - \text{CV}_{\text{WCP}}\) | ↑ | HAL 重复实验[^11] |

### 难度权重公式

\[
w_i = \frac{1}{p_i + \epsilon}, \quad p_i = \text{mean pass rate of all baselines on task } i
\]

离散化版本保留 1/2/4/8 映射，通过 pass rate 阈值自动分配。

***

## 六、防 Gaming 机制

以下三个设计共同防止系统通过"刷简单题"或"过度推断"获得虚假高分，这是对"AI Agents That Matter"和 HAL 的直接回应：[^10][^8][^11]

**机制 1：自适应权重**  
权重由 baseline pass rate 决定，系统无法在提交前预知权重，无法有针对性地刷特定类型任务。

**机制 2：难度鲁棒维度 R**  
即便系统 nWCP 很低，若 R 值低（说明只在简单任务上表现好），Leaderboard 会显示明显的雷达图偏斜，让读者一眼看出问题所在。

**机制 3：协调成本惩罚 O**  
Multi-Agent 系统若通过增加 agent 数量和轮次来微小提升精度，\(C_{\text{coord}}/C_{\text{total}}\) 增大，O 维度下降，系统整体 Profile 变差。这直接量化了 HAL 发现的"更多推理反而降低精度"现象。[^11]

***

## 七、Log 格式扩展建议

在原始 FrugalBench run log 基础上，每次运行需额外记录：

```json
{
  "run_id": "run_001",
  "seed": 42,
  "system_name": "weak-strong-router-v1",
  "task_id": "coding_debug_001",
  "score": 1.0,
  "weighted_score": 4.0,
  "difficulty": "hard",
  "pass_rate_baseline": 0.22,
  "weight_auto": 4,
  "cost": {
    "model_api_cost": 0.012,
    "tool_cost": 0.003,
    "execution_cost": 0.000,
    "coordination_cost": 0.005,
    "total_real_cost": 0.020
  },
  "repeat_run_idx": 1,
  "nwcp_this_task": 0.83
}
```

新增字段：`pass_rate_baseline`（从 baseline 运行结果自动计算）、`weight_auto`（自适应权重值）、`coordination_cost`（显式分离协调成本）、`repeat_run_idx`（支持稳定性计算）。

***

## 八、Leaderboard 展示建议

参照 HAL 的多维度展示设计，建议 Leaderboard 包含以下列：[^2]

| System | Type | nWCP ↓ | CEI ↑ | P | E | O | R | S | Verified |
|--------|------|--------|-------|---|---|---|---|---|---------|
| Router-V1 | Weak-Strong | 0.62 | 0.71 | 0.84 | 0.72 | 0.88 | 0.79 | 0.85 | ✅ |
| Debate-Agent | Multi-Agent | 1.41 | 0.38 | 0.91 | 0.17 | 0.31 | 0.55 | 0.62 | ✅ |
| Cheap-Only | Single | 1.00 | 0.00 | 0.56 | 0.50 | 1.00 | 0.48 | 0.97 | ✅ |
| Strong-Only | Single | 1.58 | 0.21 | 0.88 | 0.04 | 1.00 | 0.63 | 0.94 | ✅ |

通过此表可直观看出：高精度的 Debate-Agent 因为协调成本（O=0.31）和绝对成本效率（E=0.17）极差，整体被 Weak-Strong Router 全面碾压——这正是 FrugalBench 要揭示的核心洞察。

***

## 九、与竞品的定位对比（升级版）

| 指标/机制 | Cost-of-Pass | HAL | CostBench | RouterBench | **FrugalScore v2** |
|----------|:---:|:---:|:---:|:---:|:---:|
| 成本×精度联合评估 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 归一化（跨时间可比） | ✅（frontier） | ❌ | ❌ | ❌ | **✅（nWCP）** |
| 难度加权 | ❌ | ❌ | ❌ | ❌ | **✅（自适应）** |
| Pareto 积分指标 | ❌ | ✅（图形） | ❌ | ✅（AIQ） | **✅（CEI）** |
| 协调成本显式建模 | ❌ | 部分 | ❌ | ❌ | **✅（O 维度）** |
| 防 Gaming 机制 | ❌ | 部分 | ❌ | 部分 | **✅（3层）** |
| 多系统类型统一 | ❌ | ✅（9基准） | ❌ | ✅（路由） | **✅（全类型）** |
| 稳定性指标 | ❌ | ✅ | ❌ | ❌ | **✅（S 维度）** |
| 单一可排名标量 | ✅ | ❌ | 部分 | ✅（AIQ） | **✅（nWCP）** |

FrugalScore v2 在兼顾单一标量排名（nWCP）与多维可解释性（Profile）的同时，是目前唯一同时覆盖"协调成本建模 + 难度鲁棒 + 防 Gaming"三者的设计。

---

## References

1. [Cost-of-Pass: An Economic Framework for Evaluating Language ...](https://arxiv.org/abs/2504.13359) - We formalize cost-of-pass: the expected monetary cost of generating a correct solution. We then defi...

2. [How expensive are the best SWE-Bench agents? | Sayash Kapoor](https://www.linkedin.com/posts/ksayash_how-expensive-are-the-best-swe-bench-agents-activity-7285714117762416640-q670) - How expensive are the best SWE-Bench agents? Do reasoning models outperform language models? Can we ...

3. [RouterBench: A Benchmark for Multi-LLM Routing System - arXiv](https://arxiv.org/html/2403.12031v2) - We present RouterBench, a novel evaluation framework designed to systematically assess the efficacy ...

4. [[Revisión de artículo] CostBench: Evaluating Multi-Turn Cost ...](https://www.themoonlight.io/es/review/costbench-evaluating-multi-turn-cost-optimal-planning-and-adaptation-in-dynamic-environments-for-llm-tool-use-agents) - CostBench is a scalable, cost-centric benchmark designed to evaluate Large Language Model (LLM) agen...

5. [RouterBench: Routing Benchmark Framework](https://www.emergentmind.com/topics/routerbench) - RouterBench is a formal benchmarking framework that evaluates performance, cost efficiency, and robu...

6. [[PDF] COST-OF-PASS: AN ECONOMIC FRAMEWORK FOR EVALUATING ...](https://openreview.net/pdf?id=vC9S20zsgN) - The economics of large language models: Token allocation, fine-tuning, and optimal pricing. arXiv pr...

7. [Cost-of-Pass: An Economic Framework for Evaluating Language Models | AI Research Paper Details](https://www.aimodels.fyi/papers/arxiv/cost-pass-economic-framework-evaluating-language-models) - The widespread adoption of AI systems in the economy hinges on their ability to generate economic va...

8. [[2407.01502] AI Agents That Matter - arXiv](https://arxiv.org/abs/2407.01502) - Our analysis of current agent benchmarks and evaluation practices reveals several shortcomings that ...

9. [[Revue de papier] Holistic Agent Leaderboard: The Missing ...](https://www.themoonlight.io/fr/review/holistic-agent-leaderboard-the-missing-infrastructure-for-ai-agent-evaluation) - AI agent evaluations face significant challenges, including non-standardized, slow, and error-prone ...

10. [AI Agents That Matter: Cost-Aware Evaluation - Emergent Mind](https://www.emergentmind.com/papers/2407.01502) - This paper pioneers cost-aware evaluation for AI agents, revealing that simple baselines outperform ...

11. [The Missing Infrastructure for AI Agent Evaluation](https://arxiv.org/abs/2510.11977) - AI agents have been developed for complex real-world tasks from coding to customer service. But AI a...

