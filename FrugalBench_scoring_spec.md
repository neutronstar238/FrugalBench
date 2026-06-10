# FrugalBench 评分规范（升级版）

## 文档目的

本文档定义 FrugalBench 的升级版评分体系，包括主评分指标、可解释性指标、Weak/Strong 模型分层协议、日志格式、排行榜展示方式与防刷榜机制。FrugalBench 最初将核心问题定义为：系统为了获得单位难度加权正确分，究竟花了多少真实成本，并提出 WCP（Weighted Cost per Point）作为主指标[cite:1]。现有成本敏感评测研究进一步表明，仅用单一成本比值仍然不足，因为跨时间价格波动、不同预算点的性能形状以及基准刷榜问题都会影响结论可信度[cite:84][cite:62][cite:77]。

FrugalBench 升级版因此吸收三类代表性工作中的精华：Cost-of-Pass 提供了“前沿/基线归一化”的经济学视角[cite:84]，HAL 提供了“精度—成本 Pareto 前沿与稳定性分析”的评测基础设施视角[cite:62]，RouterBench 提供了“成本—质量曲线积分”的路由评估视角[cite:77]。在此基础上，FrugalBench 形成一套既可单值排名、又可分维解释、还能约束系统命名与防止 gaming 的完整协议。

## 设计目标

升级版评分体系应同时满足以下目标：

- **可比较**：不同时间、不同价格表、不同模型池下的结果仍可横向比较[cite:84]。
- **可排名**：必须保留单一主指标，方便论文表格、排行榜与系统对比。
- **可解释**：除主指标外，应提供对性能、成本、协调开销、难度鲁棒性和稳定性的拆解。
- **可复现**：Weak/Strong 的划分不能依赖主观命名，而必须来自公开的校准协议[cite:123][cite:124]。
- **抗刷榜**：系统不能仅靠刷简单题、增加无效反思轮次或利用不透明模型分层获得虚高排名[cite:112][cite:115]。

## 基本符号

对一个 benchmark 任务集，共有 \(N\) 个任务。对于任务 \(i\)：

- \(s_i \in [0,1]\)：系统在任务 \(i\) 上的得分，可以是 0/1，也可以是部分分。
- \(w_i > 0\)：任务 \(i\) 的难度权重。
- \(C_i\)：系统完成任务 \(i\) 的总真实成本。
- \(C_i^{\text{api}}\)：模型 API 成本。
- \(C_i^{\text{tool}}\)：工具调用成本。
- \(C_i^{\text{exec}}\)：执行成本。
- \(C_i^{\text{coord}}\)：协调成本（规划、反思、辩论、验证等）。

总加权得分与总真实成本定义为：

\[
Q = \sum_{i=1}^{N} w_i s_i
\]

\[
C_{\text{total}} = \sum_{i=1}^{N} \left(C_i^{\text{api}} + C_i^{\text{tool}} + C_i^{\text{exec}} + C_i^{\text{coord}}\right)
\]

其中协调成本被显式列为独立项，这是因为多智能体系统中的额外 planner/critic/reflection 开销往往显著，且未必带来相应性能收益[cite:1][cite:62]。

## 原始主指标：WCP

FrugalBench 原始指标定义为[cite:1]：

\[
\text{WCP} = \frac{C_{\text{total}}}{Q}
\]

WCP 越低表示系统越省钱；若 \(Q=0\)，则记为 \(+\infty\)。该定义抓住了 FrugalBench 的核心问题：比较的不是“谁最强”，而是“谁每拿到 1 单位有价值的正确分时花钱更少”[cite:1]。

但 WCP 仍然存在三类问题。第一，它是绝对货币值，价格表变化后难以比较历史结果；第二，它只给出单点值，无法描述系统在不同预算约束下的成本—性能曲线；第三，它可能被“刷简单题”的系统利用，从而掩盖对高难题和高价值任务的无能[cite:84][cite:98][cite:112]。

## 升级版主指标：nWCP

### 定义

为解决绝对价格不可比的问题，引入**归一化加权成本效率** nWCP。设 cheap baseline 系统在同一任务集和同一价格表下的总成本与加权得分分别为 \(C^{\text{cheap}}_{\text{total}}\) 与 \(Q^{\text{cheap}}\)，则：

\[
\text{nWCP} = \frac{\text{WCP}_{\text{sys}}}{\text{WCP}_{\text{cheap}}}
= \frac{C_{\text{total}} / Q}{C^{\text{cheap}}_{\text{total}} / Q^{\text{cheap}}}
\]

### 解释

- nWCP = 1：系统与 cheap baseline 具有相同成本效率。
- nWCP < 1：系统优于 cheap baseline，越低越好。
- nWCP > 1：系统成本效率劣于 cheap baseline。

这一思想与 Cost-of-Pass 中利用前沿或参照系来评价经济效率的思路一致，即不把美元绝对值当作最终结论，而是比较“相对最基本、最便宜基线”的经济增益[cite:84][cite:86]。

### 选择 cheap baseline 的原则

cheap baseline 应是一个稳定、低成本、可复现的系统，通常为单模型直答系统，且其模型来自 weak-tier 池中的候选模型。cheap baseline 不是“最弱模型”，而是“在校准集上成本最低且 ValueScore 较高的模型”，这样可避免参照系本身过于极端而失去代表性。

## 任务难度权重

### 原则

任务权重不应完全由人工主观预设，否则容易被模型和系统针对性优化。升级版建议通过 baseline 模型群的经验通过率来自动推导难度权重，这与 Cost-of-Pass 所体现的“稀缺正确解更具经济价值”是一致的[cite:84][cite:91]。

### 自动权重公式

设 \(p_i\) 为所有 baseline 系统在任务 \(i\) 上的平均通过率，则定义：

\[
 w_i = \frac{1}{p_i + \epsilon}
\]

其中 \(\epsilon\) 为平滑项，默认取 0.01。任务越难，\(p_i\) 越低，\(w_i\) 越大。

### 离散化版本

为了便于工程实现，也可将自动权重离散为四档：

\[
 w_i =
 \begin{cases}
 1, & p_i > 0.75 \\
 2, & 0.40 < p_i \le 0.75 \\
 4, & 0.15 < p_i \le 0.40 \\
 8, & p_i \le 0.15
 \end{cases}
\]

这样保留了原 FrugalBench 的 Easy/Medium/Hard/Expert 直觉，同时用真实通过率而不是手工标签来分配权重[cite:1]。

## 次级指标：CEI

### 动机

HAL 的经验表明，很多系统虽然最终精度更高，但并不总在成本上值得；Pareto 前沿比单点排行榜更能揭示“是否物有所值”[cite:62][cite:98]。RouterBench 进一步将这种成本—质量关系刻画成可积分的路由质量指标[cite:77]。因此 FrugalBench 升级版引入 **成本效率改进指数** CEI，用来补充 nWCP 的单点视角。

### 构建方式

对任一系统 \(S\)，通过改变其预算阈值、反思轮数、路由阈值、Agent 数量等超参数，可得到一组点：

\[
\{(C_{\theta_1}, Q_{\theta_1}), (C_{\theta_2}, Q_{\theta_2}), \ldots, (C_{\theta_k}, Q_{\theta_k})\}
\]

对这些点构建非递减成本—得分包络曲线 \(\widetilde{S}(c)\)。再定义 cheap baseline 曲线 \(\text{Cheap}(c)\)，以及由所有系统联合构成的 oracle 前沿 \(\text{Oracle}(c)\)。

### 公式

\[
\text{CEI}(S) = \frac{\int_{c_{\min}}^{c_{\max}} [\widetilde{S}(c) - \text{Cheap}(c)] \, dc}{\int_{c_{\min}}^{c_{\max}} [\text{Oracle}(c) - \text{Cheap}(c)] \, dc}
\]

### 解释

- CEI = 0：系统整体效率与 cheap baseline 无差别。
- CEI = 1：系统达到了当前候选池中的 oracle 前沿。
- CEI < 0：系统在整个预算区间上都不如 cheap baseline。

CEI 不是主排行榜指标，但它提供了系统在“预算受限情景”下是否真正有价值的证据，这一点对多智能体系统尤其关键[cite:62][cite:77]。

## 五维可解释性 Profile

单一指标不足以解释系统为什么高分或低分。升级版 FrugalBench 定义五维解释向量 \((P,E,O,R,S)\)，每一维均归一化到 [0,1] 且“越高越好”。

### P：Performance

\[
P = \frac{\sum_i w_i s_i}{\sum_i w_i}
\]

P 表示难度加权后的平均正确性，是系统“会不会做”的直接体现。

### E：Efficiency

\[
E = 1 - \text{clip}(\text{nWCP}, 0, 1)
\]

实际实现中，也可写为一个平滑函数，例如 \(E = 1/(1+\text{nWCP})\)，但推荐保持与 nWCP 语义对应，便于读者理解。E 越高表示系统越省钱。

### O：Overhead Penalty

\[
O = 1 - \frac{C_{\text{coord}}}{C_{\text{total}}}
\]

其中 \(C_{\text{coord}} = \sum_i C_i^{\text{coord}}\)。若系统大量成本花在 planner、反思、辩论和验证上，而不是直接解决任务，则 O 会下降。HAL 的大规模实验已经说明，多出来的 reasoning effort 很多时候并不转化为更高精度[cite:62]。

### R：Robustness Across Difficulty

设 \(r_d\) 表示系统在每个难度层级 \(d\) 上的单位成本效率（可用分层 nWCP 或分层得分/成本比表示），则：

\[
R = 1 - \frac{\sigma(\{r_d\})}{\tau_R}
\]

其中 \(\sigma\) 为标准差，\(\tau_R\) 为归一化常数，默认可设为训练期经验上限。R 高意味着系统在 Easy 到 Expert 各层上都较均衡，不会只在简单题上特别好、在难题上完全崩溃。该维度专门用于防止“刷简单题”获得表面高分[cite:112][cite:115]。

### S：Stability

设同一系统在不同随机种子下重复运行 \(m\) 次，每次得到一个 WCP 值，则：

\[
S = 1 - \min\left(1, \text{CV}(\text{WCP}_1, \ldots, \text{WCP}_m)\right)
\]

其中 CV 为变异系数。S 高意味着系统成本效率稳定，结论不依赖偶然随机性。HAL 强调了 agent 评测中重复实验与稳定性的重要性，因为一次高分往往并不代表稳定高分[cite:62]。

## 总体展示方式

主排行榜按 nWCP 升序排序，同时展示 CEI 与五维 Profile。建议表头如下：

| System | Type | nWCP ↓ | CEI ↑ | P | E | O | R | S | Verified |
|--------|------|--------|-------|---|---|---|---|---|----------|
| Router-V1 | Weak-Strong | 0.62 | 0.71 | 0.84 | 0.72 | 0.88 | 0.79 | 0.85 | ✅ |
| Debate-Agent | Multi-Agent | 1.41 | 0.38 | 0.91 | 0.17 | 0.31 | 0.55 | 0.62 | ✅ |
| Cheap-Only | Single | 1.00 | 0.00 | 0.56 | 0.50 | 1.00 | 0.48 | 0.97 | ✅ |
| Strong-Only | Single | 1.58 | 0.21 | 0.88 | 0.04 | 1.00 | 0.63 | 0.94 | ✅ |

该表中的示例数值仅用于展示字段结构，不构成真实实验结果。真实提交必须同时上传原始日志、价格配置、模型版本和随机种子。

## Weak / Strong 模型分层协议

### 原则

FrugalBench 中的 `Weak-Strong` 表示的是一种系统架构类型，而不是对某个具体模型品牌的主观命名。FrugalGPT、RouteLLM 等工作都建立在“更便宜但较弱的模型负责简单请求，更贵且更强的模型负责困难请求”的范式上[cite:23][cite:123][cite:124]。因此，FrugalBench 规定 weak-tier 与 strong-tier 必须通过公开的校准协议自动确定，而不能由提交者自由命名。

### Calibration Suite

Benchmark 维护方维护一个小规模、公开的 Calibration Suite，用于给候选模型打“成本—能力”标签。建议至少覆盖以下能力维度：

- 通用知识 / QA
- 数学与推理
- 编码与代码生成
- 工具使用或轻量 Agent 任务

Calibration Suite 的规模应足够小，以支持新模型快速纳入，但要覆盖 FrugalBench 主任务分布中的核心能力。所有想参与 FrugalBench 的候选模型都必须先完成该套校准评测。

### 候选模型的成本—能力点

对每个候选模型 \(M\)，在 Calibration Suite 上计算：

- 加权准确率 \(A_M\)
- 总真实成本 \(C_M\)

并定义：

\[
\text{ValueScore}(M) = \frac{A_M}{C_M}
\]

其中 \(A_M\) 体现模型能力，\(C_M\) 体现成本。ValueScore 不是最终 tier 划分的唯一依据，但可帮助选择 cheap baseline 和筛选低价值模型。

### Cheap baseline 的选择

设 \(M_{\text{cheap}}\) 为成本最低且 ValueScore 排名前列的候选模型，则定义其为 cheap baseline。cheap baseline 将被用于：

- nWCP 的归一化参照；
- weak-tier 的候选筛选；
- `Cheap-Only` 系统类型的单模型基线。

### Weak-tier 定义

模型 \(M\) 属于 weak-tier，当且仅当同时满足：

\[
C_M \le \alpha \cdot C_{\text{cheap}}
\]

\[
A_M \ge \beta \cdot A_{\max}
\]

其中默认 \(\alpha = 1.5\)，\(\beta = 0.6\)，\(A_{\max}\) 为候选模型中最高的校准准确率。也就是说，weak-tier 模型必须足够便宜，但不能弱到失去“处理简单任务”的基本能力。

### Strong-tier 定义

模型 \(M\) 属于 strong-tier，当且仅当同时满足：

\[
A_M \ge A_{\text{cheap}} + \Delta A
\]

并且其成本位于候选模型成本分布的中上区间，默认可要求在上 50% 分位以上。这里 \(\Delta A\) 默认为 10% 绝对精度提升。其意图是保证 strong-tier 模型既“明显更强”，又“确实更贵”，从而符合 weak-strong routing 的经济学设定[cite:124][cite:127]。

### 参数更新与版本化

参数 \(\alpha\)、\(\beta\)、\(\Delta A\) 可以随模型生态变化而调整，但任何改动必须伴随 benchmark 版本号更新，并保留旧版本校准结果。这样可以确保历史结果可追溯、可复算、可比较。

## SystemType 字段定义

在排行榜和日志中，`SystemType` 字段用于描述系统架构，而不是单纯描述用了哪个模型：

- `Single`：仅调用单一模型，不做路由，不做多智能体协调。
- `Weak-Strong`：在 weak-tier 模型 \(M_w\) 与 strong-tier 模型 \(M_s\) 之间进行路由或级联。
- `Multi-Agent`：存在多个角色、多个轮次或多个子代理的显式协调过程。
- `Router-Based`：在多个模型、工具或工作流之间进行选择，但不一定严格是二元 weak/strong 结构。
- `Hybrid`：兼具以上多种特征。

其中：

- `Cheap-Only` 指只调用 \(M_{\text{cheap}}\) 的 `Single` 系统；
- `Strong-Only` 指只调用 strong-tier 中校准准确率最高模型的 `Single` 系统；
- `Router-V1` 若标记为 `Weak-Strong`，则必须在提交说明中明确写出其 \(M_w\) 与 \(M_s\) 分别是谁，且二者必须来自校准协议定义的 weak-tier 和 strong-tier。

## 成本定义

FrugalBench 的总真实成本由四部分构成[cite:1]：

\[
C_{\text{total}} = C_{\text{api}} + C_{\text{tool}} + C_{\text{exec}} + C_{\text{coord}}
\]

各部分定义如下：

- **API Cost**：输入 token、输出 token、图像/音频 token 等模型供应商计费项。
- **Tool Cost**：外部搜索、数据库、代码执行、检索服务、付费 API 等直接调用成本。
- **Execution Cost**：运行容器、GPU、CPU 时间、沙盒环境等成本。
- **Coordination Cost**：planner、critic、reflection、debate、verification、voting 等引入的额外调用与协调开销。

对于本地模型，也应通过统一价格表将 GPU/CPU 时间折算为成本，否则本地部署系统会被不公平地视为“零成本”。

## 防 Gaming 机制

### 机制一：自动难度权重

任务难度由 baseline 通过率自动导出，而不是提交者已知的固定标签。这使得系统难以针对某一类“低成本高收益”的题目进行过拟合。

### 机制二：难度鲁棒性维度 R

即使一个系统 nWCP 很低，只要它在 Expert 难度层上的效率远低于 Easy 层，R 就会显著下降。这样排行榜不会只展示单个漂亮数字，而会明确暴露“只会做简单题”的系统结构。

### 机制三：协调开销维度 O

多智能体系统若靠增加 agent、增加投票轮数、增加自反思步骤来换取微弱精度提升，其 \(C_{\text{coord}} / C_{\text{total}}\) 会快速上升，从而降低 O。该设计直接吸收了 HAL 对“更多推理不一定更好”的经验结论[cite:62]。

### 机制四：重复实验与 S

所有正式排行榜结果都应至少重复 3 次，并报告均值与标准差。若只报单次最优结果，则系统可以通过 sampling luck 获得虚高排名，这与稳定部署目标相悖[cite:62]。

## 日志格式

每次运行都必须生成结构化日志，建议格式如下：

```json
{
  "run_id": "run_001",
  "seed": 42,
  "system_name": "weak_strong_router_v1",
  "system_type": "Weak-Strong",
  "task_id": "coding_debug_001",
  "score": 1.0,
  "weighted_score": 4.0,
  "difficulty": "hard",
  "baseline_pass_rate": 0.22,
  "weight_auto": 4,
  "model_assignment": {
    "weak_model": "qwen-small",
    "strong_model": "gpt-4-class"
  },
  "cost": {
    "model_api_cost": 0.012,
    "tool_cost": 0.003,
    "execution_cost": 0.000,
    "coordination_cost": 0.005,
    "total_real_cost": 0.020
  },
  "usage": {
    "input_tokens": 3200,
    "output_tokens": 900,
    "model_calls": 4,
    "tool_calls": 2
  },
  "repeat_run_idx": 1,
  "final_answer": "...",
  "grading_result": 1.0
}
```

新增字段 `baseline_pass_rate`、`weight_auto`、`coordination_cost`、`repeat_run_idx` 与 `model_assignment` 对升级版评分至关重要。前两者支撑自动难度权重，第三项支撑 O 维度，第四项支撑稳定性 S，第五项支撑 Weak/Strong 可复现性。

## 结果提交流程

为了支持 self-reported、reproduced 和 verified 三类状态，提交者至少应同时上传：

- 系统说明文档；
- 模型名称与版本；
- prompt 模板；
- 工具配置；
- pricing.yaml；
- 原始运行日志；
- 评测输出文件；
- 若为 Weak-Strong 系统，还需附上 Calibration Suite 上的模型分层证明。

这与 HAL 对 agent evaluation 基础设施标准化的方向一致：性能、成本和日志都应被完整记录，才能支持可信比较[cite:62][cite:98]。

## 推荐基线

初版 FrugalBench 至少应包含以下基线：

- Cheap-Only
- Strong-Only
- Single + Self-Reflection
- Single + Retry
- Weak-Strong Router
- Confidence-Based Router
- Planner-Solver
- Solver-Critic
- Multi-Agent Debate
- Verifier-Based Agent

这些基线覆盖了从单模型到路由、再到多智能体的主要系统结构，与 FrugalBench 最初的任务定位一致[cite:1]。其中 Cheap-Only 与 Strong-Only 不只是最低和最高档模型，更是 nWCP 归一化与 Weak/Strong 分层协议中的基础锚点。

## 建议的论文表述方式

若 FrugalBench 写成论文，建议在方法章节中将本规范浓缩成三部分：

1. **Definition of Real-Cost Efficiency**：定义 WCP 与 nWCP；
2. **Definition of Weak/Strong Tiering Protocol**：定义 Calibration Suite 和模型分层；
3. **Interpretability Profile**：定义 P/E/O/R/S 五维向量。

附录中再提供：

- Calibration Suite 具体样本；
- \(\alpha, \beta, \Delta A\) 的灵敏度分析；
- CEI 曲线积分计算细节；
- 各日志字段与 schema。

这样主文可突出理论贡献，附录保障可复现性，也更符合 benchmark 论文的写作方式。

## 实施建议

若采用 MVP 路线，建议按以下顺序实现：

1. 先实现原始 WCP 和成本跟踪器；
2. 再实现 cheap baseline 与 nWCP；
3. 加入自动难度权重；
4. 扩展日志格式，支持 coordination cost 与 repeated runs；
5. 最后实现 CEI 与五维可解释性展示。

这样可以先快速产出一个可用 benchmark，再逐步把说服力强的高级机制补齐。对投稿而言，哪怕首版只完整实现 nWCP + Weak/Strong tiering + O/R/S 三维解释，也已经比纯 accuracy leaderboard 更具新意和现实价值[cite:11][cite:62]。

## 结语

FrugalBench 的核心价值不在于重新发明一个“谁最准”的排行榜，而在于把“真实部署里的单位价值成本”作为主角[cite:1]。升级版评分规范通过引入 nWCP、CEI、五维解释向量与 Weak/Strong 模型分层协议，将这一理念从一个简单指标扩展为一套可比较、可解释、可复现、可防刷榜的完整 benchmark 协议。对于单模型、路由系统、多智能体系统和 hybrid system，这套协议都能更准确地回答同一个问题：**哪个系统真正做到了 spend less, solve more**。[cite:1][cite:84][cite:62]
