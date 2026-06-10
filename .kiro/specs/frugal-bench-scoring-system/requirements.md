# Requirements Document: FrugalBench Scoring System

## Introduction

FrugalBench is a cost-aware AI system evaluation benchmark designed to address critical weaknesses in existing benchmarks that focus solely on accuracy. The system implements a comprehensive three-layer scoring architecture to evaluate AI agents and LLM-based systems through the lens of cost-efficiency, answering the fundamental question: "Which system truly achieves 'spend less, solve more'?" This requirements document specifies the functional and non-functional requirements for FrugalBench, derived from the technical design that incorporates lessons learned from Cost-of-Pass, HAL, RouterBench, and CostBench research.

The system must support evaluation of diverse AI system architectures including single-model systems, weak-strong routing systems, multi-agent systems, router-based systems, and hybrid systems. It must provide cross-temporal comparability through normalization, interpretability through multi-dimensional profiles, and resistance to gaming through automatic difficulty weighting and robustness metrics.

## Glossary

- **System**: An AI agent or LLM-based system being evaluated by FrugalBench
- **Task**: An individual evaluation problem with ground truth and scoring function
- **Calibration_Suite**: A small, standardized set of tasks used to classify models into tiers
- **Model**: A large language model API or local model used by systems
- **Weak_Tier**: Models classified as low-cost with acceptable capability
- **Strong_Tier**: Models classified as high-capability with higher cost
- **Cheap_Baseline**: The lowest-cost model with high ValueScore, used for normalization
- **Baseline_System**: A set of reference systems used to compute task pass rates
- **Pass_Rate**: The average success rate of baseline systems on a task
- **Weight**: The difficulty weight assigned to a task based on pass rate
- **Score**: A value in [0,1] representing system performance on a task
- **Cost**: Real monetary cost including API, tool, execution, and coordination costs
- **WCP**: Weighted Cost per Point (total cost divided by weighted score)
- **nWCP**: Normalized WCP (system WCP divided by cheap baseline WCP)
- **CEI**: Cost Efficiency Improvement Index (integral-based metric vs Pareto frontier)
- **Profile**: Five-dimensional interpretability vector (P, E, O, R, S)
- **Leaderboard**: Public ranking of systems by nWCP with verification status
- **Verification_Status**: Classification of submission trustworthiness (self-reported, reproduced, verified)
- **Run**: A single execution of a system across the task suite
- **Pricing_Table**: Versioned cost configuration for models, tools, and execution
- **Coordination_Cost**: Cost of meta-cognitive operations (planning, reflection, debate, verification)

## Requirements

### Requirement 1: Model Calibration and Tiering

**User Story:** As a benchmark administrator, I want to automatically classify candidate models into weak-tier and strong-tier categories based on cost-accuracy calibration, so that system types are defined by objective metrics rather than subjective naming.

#### Acceptance Criteria

1. WHEN a new model is added to the candidate pool, THE Calibration_Suite SHALL execute calibration tasks across QA, math, coding, and tool-use dimensions
2. WHEN calibration completes for a model, THE System SHALL compute accuracy A_M, total cost C_M, and ValueScore = A_M / C_M
3. THE System SHALL select the cheap baseline as the model with minimum cost among candidates meeting minimum accuracy thresholds, where minimum cost is the actual lowest cost value in the candidate pool
4. WHEN assigning weak-tier classification, THE System SHALL verify that model cost ≤ α × C_cheap AND accuracy ≥ β × A_max, where α=1.5 and β=0.6 are default parameters
5. WHEN assigning strong-tier classification, THE System SHALL verify that model accuracy ≥ A_cheap + ΔA AND cost is in the upper 50th percentile, where ΔA=0.1 is the default threshold
6. THE System SHALL exclude models that meet neither weak-tier nor strong-tier criteria from routing system eligibility
7. WHEN calibration parameters (α, β, ΔA) are updated, THE System SHALL increment the benchmark version number and archive previous calibration results
8. THE Calibration_Suite SHALL complete execution for a single model in under 10 minutes

### Requirement 2: Automatic Difficulty Weighting

**User Story:** As a benchmark administrator, I want task difficulty weights to be derived automatically from baseline system pass rates, so that systems cannot game the benchmark by optimizing for pre-known difficulty labels.

#### Acceptance Criteria

1. WHEN baseline systems complete evaluation, THE Difficulty_Weight_Engine SHALL compute pass rate p_i for each task i as the average success rate across all baseline systems
2. WHEN computing continuous weights, THE System SHALL apply the formula w_i = 1 / (p_i + ε) where ε = 0.01 is the smoothing parameter
3. WHEN computing discrete weights, THE System SHALL assign w_i = 1 if p_i > 0.75, w_i = 2 if 0.40 < p_i ≤ 0.75, w_i = 4 if 0.15 < p_i ≤ 0.40, and w_i = 8 if p_i ≤ 0.15, where boundary values are assigned to the higher difficulty level
4. IF pass rate p_i = 0, THEN THE System SHALL assign weight w_i = 1/ε to prevent infinite weights
5. IF pass rate p_i = 1, THEN THE System SHALL assign weight w_i = 1/(1+ε) to ensure non-zero weight
6. THE System SHALL version and archive weight assignments with each benchmark version
7. WHEN a task suite is updated, THE System SHALL recompute all baseline pass rates and weights before accepting new system submissions, and SHALL block new submissions until recomputation completes

### Requirement 3: Cost Tracking and Decomposition

**User Story:** As a system evaluator, I want all costs to be tracked and decomposed into four categories (API, tool, execution, coordination), so that I can understand the cost structure of different system architectures.

#### Acceptance Criteria

1. WHEN a model API call is made, THE Cost_Tracker SHALL record input tokens, output tokens, model name, and compute cost using the current pricing table
2. WHEN an external tool is invoked, THE Cost_Tracker SHALL record tool name, invocation count, and compute cost based on tool pricing
3. WHEN a system executes, THE Cost_Tracker SHALL monitor execution time and compute GPU/CPU/memory costs using execution pricing
4. WHEN a coordination operation occurs (planner, reflection, debate, verification, voting, critic), THE Cost_Tracker SHALL separately track it as coordination cost
5. THE System SHALL compute total cost as C_total = C_api + C_tool + C_exec + C_coord for each task
6. THE System SHALL aggregate costs per task and per run with full breakdown
7. WHEN a local model is used, THE System SHALL convert GPU/CPU time to equivalent cost using the pricing table
8. THE Cost_Tracker SHALL ensure all cost values are non-negative in final computations, but MAY allow temporary negative intermediate values during calculation

### Requirement 4: Score Aggregation and Weighting

**User Story:** As a benchmark evaluator, I want task scores to be aggregated with difficulty weights, so that systems are rewarded for solving harder problems.

#### Acceptance Criteria

1. WHEN a system completes a task, THE Score_Aggregator SHALL record raw score s_i ∈ [0,1] based on the task evaluator
2. THE System SHALL compute weighted score for task i as w_i × s_i where w_i is the difficulty weight
3. THE System SHALL compute total weighted score Q = Σ(w_i × s_i) across all tasks in a run
4. THE System SHALL partition scores by difficulty level (Easy, Medium, Hard, Expert) for robustness analysis
5. THE System SHALL support partial scoring with continuous values in [0,1], not just binary 0/1
6. IF a task fails, times out, or does not complete execution, THE System SHALL assign score s_i = 0 for that task
7. THE System SHALL handle missing tasks by excluding them from both numerator and denominator in average calculations
8. THE System SHALL reject evaluation runs where all difficulty weights sum to zero

### Requirement 5: nWCP Calculation (Layer 1)

**User Story:** As a benchmark user, I want a primary ranking metric that is normalized against a cheap baseline, so that results are comparable across different time periods and price changes.

#### Acceptance Criteria

1. THE System SHALL compute WCP for a system as WCP_sys = C_total / Q where C_total is total cost and Q is total weighted score
2. THE System SHALL retrieve the cheap baseline's WCP as WCP_cheap = C_cheap_total / Q_cheap from calibration results
3. THE System SHALL compute normalized WCP as nWCP = WCP_sys / WCP_cheap
4. IF weighted score Q = 0, THEN THE System SHALL assign nWCP = +∞ to indicate zero productivity, and SHALL treat +∞ as satisfying the non-negative requirement
5. THE System SHALL validate that all intermediate values (C_total, Q, WCP_sys, WCP_cheap) are non-negative before computing nWCP
6. THE System SHALL interpret nWCP < 1.0 as better efficiency than cheap baseline
7. THE System SHALL interpret nWCP > 1.0 as worse efficiency than cheap baseline
8. THE System SHALL validate that nWCP is non-negative or +∞ in all cases, and SHALL prevent negative nWCP values through validation checks
9. THE Leaderboard SHALL rank systems by nWCP in ascending order (lower is better)

### Requirement 6: CEI Calculation (Layer 2)

**User Story:** As a researcher, I want to understand how a system's cost-quality tradeoff compares to the Pareto frontier, so that I can assess whether higher costs are justified by performance gains.

#### Acceptance Criteria

1. WHEN a system provides multiple hyperparameter configurations, THE CEI_Calculator SHALL collect (cost, score) points across configurations
2. THE System SHALL build a non-decreasing envelope curve S̃(c) from system points where each cost c maps to the maximum achievable score at or below that cost level
3. THE System SHALL construct the cheap baseline curve Cheap(c) as a reference lower bound
4. THE System SHALL construct the oracle frontier Oracle(c) from Pareto-optimal points across all evaluated systems
5. THE System SHALL compute CEI numerator as the integral ∫[S̃(c) - Cheap(c)]dc × (c_max - c_min) over the cost range [c_min, c_max]
6. THE System SHALL compute CEI denominator as the integral ∫[Oracle(c) - Cheap(c)]dc over the same cost range
7. THE System SHALL compute CEI = numerator / denominator and clamp the result to [0, 1]
8. IF CEI = 0, THE System SHALL interpret this as no improvement over cheap baseline
9. IF CEI = 1, THE System SHALL interpret this as reaching the oracle frontier
10. THE System SHALL use trapezoidal rule or adaptive quadrature for integral computation

### Requirement 7: Performance Dimension (P)

**User Story:** As a system developer, I want to see the difficulty-weighted average accuracy of my system, so that I can understand its capability independent of cost.

#### Acceptance Criteria

1. THE Profile_Calculator SHALL compute Performance as P = Σ(w_i × s_i) / Σw_i across all tasks
2. THE System SHALL normalize P to the range [0, 1] where 1 represents perfect weighted accuracy
3. THE System SHALL interpret higher P values as better overall capability
4. IF no tasks are completed successfully, THE System SHALL assign P = 0

### Requirement 8: Efficiency Dimension (E)

**User Story:** As a system developer, I want to see cost efficiency as a normalized metric, so that I can compare my system's economic performance to the baseline.

#### Acceptance Criteria

1. THE Profile_Calculator SHALL compute Efficiency as E = 1 - min(nWCP, 1.0)
2. THE System SHALL clamp E to the range [0, 1] where 1 represents maximum efficiency
3. THE System SHALL interpret higher E values as better cost efficiency
4. IF nWCP ≥ 1.0, THE System SHALL clamp Efficiency to 0% to indicate worse-than-baseline performance

### Requirement 9: Overhead Dimension (O)

**User Story:** As a system developer, I want to understand what fraction of my system's cost is spent on coordination rather than task solving, so that I can optimize meta-cognitive operations.

#### Acceptance Criteria

1. THE Profile_Calculator SHALL compute Overhead as O = 1 - (C_coord / C_total) where C_coord is total coordination cost
2. THE System SHALL normalize O to the range [0, 1] where 1 represents zero coordination overhead
3. THE System SHALL interpret higher O values as more efficient resource allocation
4. IF C_total = 0, THE System SHALL assign O = 1 by default
5. THE System SHALL separately track planner, reflection, debate, verification, voting, and critic operations as coordination costs

### Requirement 10: Robustness Dimension (R)

**User Story:** As a benchmark evaluator, I want to detect systems that only perform well on easy tasks, so that I can identify gaming behavior through difficulty-specific analysis.

#### Acceptance Criteria

1. THE Profile_Calculator SHALL partition tasks into difficulty levels (Easy, Medium, Hard, Expert) based on discrete weights
2. THE System SHALL compute per-difficulty efficiency r_d for each difficulty level d
3. THE System SHALL compute the standard deviation σ of efficiency values across difficulty levels
4. THE Profile_Calculator SHALL compute Robustness as R = 1 - σ(r_d) / τ_R where τ_R = 0.5 is the normalization constant
5. THE System SHALL normalize R to the range [0, 1] where 1 represents perfect consistency across difficulties
6. THE System SHALL interpret higher R values as better robustness against difficulty variations
7. IF standard deviation σ(r_d) ≥ τ_R, THE System SHALL clamp R to 0
8. WHEN only one difficulty level contains tasks, THE System SHALL use normal robustness calculation instead of assigning default values
9. IF no tasks exist in the evaluation, THE System SHALL report an error or undefined state for robustness

### Requirement 11: Stability Dimension (S)

**User Story:** As a benchmark administrator, I want to measure the variance in system performance across repeated runs, so that I can detect unreliable systems that depend on random luck.

#### Acceptance Criteria

1. THE System SHALL require minimum 3 repeated runs for each system submission to the leaderboard
2. WHEN computing stability, THE Profile_Calculator SHALL collect WCP values from all repeated runs
3. THE System SHALL compute coefficient of variation as CV = σ(WCP) / mean(WCP) where σ is standard deviation
4. THE Profile_Calculator SHALL compute Stability as S = 1 - min(CV, 1.0)
5. THE System SHALL normalize S to the range [0, 1] where 1 represents perfect stability
6. THE System SHALL interpret higher S values as more reliable and consistent performance
7. IF CV ≥ 1.0, THE System SHALL clamp S to 0 to indicate high variance
8. WHEN CV = 0.0, THE System SHALL assign S = 1.0 regardless of normalization formula to ensure perfect stability representation

### Requirement 12: System Execution with Repeated Runs

**User Story:** As a system evaluator, I want to execute systems multiple times with different random seeds, so that I can measure performance stability and avoid misleading single-run results.

#### Acceptance Criteria

1. WHEN a system is evaluated, THE Execution_Engine SHALL execute the system at least 3 times with different random seeds
2. THE System SHALL record the seed value for each run to ensure reproducibility
3. THE System SHALL assign a unique run ID to each execution
4. THE System SHALL generate structured JSON logs for each task execution within a run
5. THE Execution_Engine SHALL aggregate costs and scores across all tasks in a run
6. THE System SHALL compute mean and standard deviation of WCP across all repeated runs
7. WHEN execution fails or times out, THE System SHALL record the failure and continue with remaining runs in all cases
8. THE System SHALL enforce strict execution time limits of 1 hour per run regardless of task count

### Requirement 13: Structured Logging

**User Story:** As a benchmark administrator, I want all evaluation runs to generate structured logs, so that results can be reproduced and verified independently.

#### Acceptance Criteria

1. THE Execution_Engine SHALL generate a JSON log entry for each task execution
2. THE log SHALL include run_id, seed, repeat_run_idx, system_name, system_type, task_id, score, weighted_score, difficulty, weight, baseline_pass_rate
3. THE log SHALL include cost breakdown with api_cost, tool_cost, execution_cost, coordination_cost, and total_cost
4. THE log SHALL include usage metrics with input_tokens, output_tokens, model_call_count, tool_call_count
5. THE log SHALL include model_assignment for Weak-Strong systems specifying weak_model and strong_model
6. THE log SHALL include coordination_steps array listing all planner, reflection, debate, and verification operations
7. THE log SHALL include execution_time_ms, final_answer, and grading_result for each task
8. THE System SHALL ensure all log entries follow a versioned schema for forward compatibility

### Requirement 14: Leaderboard Generation and Ranking

**User Story:** As a benchmark user, I want to see a ranked leaderboard with primary and secondary metrics, so that I can compare systems and understand their relative strengths.

#### Acceptance Criteria

1. THE Leaderboard_Generator SHALL rank systems by nWCP in ascending order (lower is better)
2. THE Leaderboard SHALL display system_name, system_type, nWCP, CEI, and all five profile dimensions (P, E, O, R, S)
3. THE Leaderboard SHALL display verification_status for each entry (self-reported, reproduced, verified)
4. WHEN two systems have identical nWCP values, THE System SHALL use CEI as a tiebreaker (higher is better)
5. THE Leaderboard SHALL support filtering by system_type (Single, Weak-Strong, Multi-Agent, Router-Based, Hybrid)
6. THE Leaderboard SHALL support filtering by verification_status
7. THE System SHALL export leaderboard data in both JSON and markdown formats
8. THE Leaderboard SHALL include generation timestamp and benchmark version number

### Requirement 15: Verification Status Management

**User Story:** As a benchmark administrator, I want to track the verification status of submissions, so that users can distinguish between unverified claims and independently reproduced results.

#### Acceptance Criteria

1. WHEN a system is first submitted, THE System SHALL assign verification_status = "self-reported"
2. WHEN a submission is reproduced by an independent party using provided logs and configuration, THE System SHALL allow status update to "reproduced" only after verification of actual reproduction with proof
3. WHEN a submission is verified by benchmark maintainers through complete re-execution, THE System SHALL allow status update to "verified" only after confirming actual verification results
4. THE System SHALL support automatic status updates when reproduction or verification is programmatically confirmed
5. THE System SHALL display self-reported submissions with ❌ icon, reproduced with ⚠️ icon, and verified with ✅ icon
6. THE Leaderboard SHALL clearly indicate that self-reported results have not been independently validated
7. THE System SHALL require submission of pricing table, model versions, prompt templates, and original logs for verification eligibility

### Requirement 16: Pricing Table Management

**User Story:** As a benchmark administrator, I want to maintain versioned pricing tables for models, tools, and execution resources, so that cost calculations remain accurate and historical results can be reproduced.

#### Acceptance Criteria

1. THE System SHALL maintain a pricing table with per-token costs for all supported models
2. THE pricing table SHALL include input_cost_per_token, output_cost_per_token, and minimum_cost for each model
3. THE pricing table SHALL include per-call or per-unit costs for all supported tools
4. THE pricing table SHALL include GPU cost per hour by GPU type, CPU cost per hour, and memory cost per GB-hour
5. WHEN the pricing table is updated, THE System SHALL increment the version number and archive the previous version
6. THE System SHALL associate each evaluation run with the specific pricing table version used
7. THE System SHALL support cost recalculation for historical runs using archived pricing tables
8. THE System SHALL validate that all pricing values are non-negative

### Requirement 17: System Type Classification

**User Story:** As a system developer, I want my system to be classified by architecture type rather than model names, so that comparisons are based on structural patterns rather than brand names.

#### Acceptance Criteria

1. THE System SHALL classify systems into types: Single, Weak-Strong, Multi-Agent, Router-Based, or Hybrid
2. WHEN a system uses only one model without routing or coordination, THE System SHALL classify it as Single
3. WHEN a system routes between weak-tier and strong-tier models, THE System SHALL classify it as Weak-Strong and require model_assignment specification
4. WHEN a system involves multiple agents with explicit coordination rounds, THE System SHALL classify it as Multi-Agent
5. WHEN a system routes between multiple models, tools, or workflows, THE System SHALL classify it as Router-Based
6. WHEN a system combines characteristics of multiple architecture types, THE System SHALL classify it as Hybrid
7. THE System SHALL validate that Weak-Strong systems specify models from calibrated weak-tier and strong-tier pools
8. THE Leaderboard SHALL display system_type for all entries

### Requirement 18: Anti-Gaming Through Robustness Metrics

**User Story:** As a benchmark designer, I want to detect and penalize systems that only perform well on easy tasks, so that gaming through simple-task optimization is disincentivized.

#### Acceptance Criteria

1. THE System SHALL compute per-difficulty-level efficiency for Easy, Medium, Hard, and Expert tasks
2. WHEN a system shows high variance in efficiency across difficulty levels, THE System SHALL assign a low Robustness (R) score
3. THE Leaderboard SHALL display the R dimension prominently to make difficulty-specific weaknesses visible
4. THE System SHALL expose radar charts showing all five profile dimensions to visualize gaming patterns
5. WHEN a system has nWCP < 1 but R < 0.5, THE System SHALL flag this as potential gaming behavior only if the system shows selective performance patterns indicating intentional optimization for easy tasks

### Requirement 19: Anti-Gaming Through Coordination Overhead Tracking

**User Story:** As a benchmark evaluator, I want to detect systems that add unnecessary coordination operations without proportional performance gains, so that excessive meta-reasoning is discouraged.

#### Acceptance Criteria

1. THE System SHALL separately track all planner, reflection, debate, verification, voting, and critic operations as coordination costs
2. WHEN coordination cost exceeds 30% of total cost, THE System SHALL assign overhead dimension O < 0.7
3. THE Leaderboard SHALL display the O dimension to make coordination inefficiency visible
4. THE System SHALL support analysis of coordination cost vs performance gain for multi-agent systems
5. WHEN a system increases coordination cost without proportional performance improvement, THE System SHALL reflect this in decreased O and E scores

### Requirement 20: Anti-Gaming Through Mandatory Stability Reporting

**User Story:** As a benchmark administrator, I want to require multiple repeated runs, so that systems cannot report only their best single-run result.

#### Acceptance Criteria

1. THE System SHALL reject leaderboard submissions with fewer than 3 repeated runs
2. THE System SHALL report mean nWCP, standard deviation, and coefficient of variation for all submissions
3. WHEN a system shows CV > 0.3 in WCP across runs, THE System SHALL assign stability dimension S < 0.7
4. THE Leaderboard SHALL display the S dimension to make instability visible
5. THE System SHALL support filtering to show only systems with S > threshold for production-readiness analysis

### Requirement 21: Baseline System Execution

**User Story:** As a benchmark administrator, I want to execute a set of baseline systems to compute task pass rates and establish reference points, so that difficulty weights and normalization values are empirically grounded.

#### Acceptance Criteria

1. THE System SHALL maintain a set of baseline systems including Cheap-Only, Strong-Only, Single+Reflection, Weak-Strong Router, and Multi-Agent Debate
2. WHEN a new benchmark version is released, THE System SHALL execute all baseline systems across the full task suite
3. THE System SHALL compute pass rates from baseline results for automatic difficulty weighting
4. THE System SHALL establish cheap baseline WCP value for normalization
5. THE System SHALL contribute baseline results to the oracle frontier for CEI calculation
6. THE baseline execution SHALL complete within 1 hour for a 1000-task suite

### Requirement 22: Model Assignment Tracking for Routing Systems

**User Story:** As a routing system developer, I want to declare which weak and strong models my system uses, so that reviewers can verify tier compliance and understand routing behavior.

#### Acceptance Criteria

1. WHEN a system is classified as Weak-Strong, THE System SHALL require model_assignment declaration with weak_model and strong_model fields
2. THE System SHALL validate that weak_model is from the calibrated weak-tier pool
3. THE System SHALL validate that strong_model is from the calibrated strong-tier pool
4. THE System SHALL reject submissions that claim Weak-Strong type without valid model_assignment
5. THE structured log SHALL record model_assignment for every run to enable post-hoc analysis
6. THE System SHALL support router-based systems with multiple models by recording all models used

### Requirement 23: Historical Reproducibility

**User Story:** As a researcher, I want to reproduce historical benchmark results even after API prices change, so that I can verify published claims and perform longitudinal studies.

#### Acceptance Criteria

1. THE System SHALL archive all pricing tables with version numbers and timestamps
2. THE System SHALL associate each run with its pricing table version in structured logs
3. WHEN reproducing a historical result, THE System SHALL use the archived pricing table from the original evaluation
4. THE System SHALL store config_hash for each run to detect configuration drift
5. THE System SHALL enable cost recalculation for any historical run using current pricing tables for comparison
6. THE System SHALL maintain benchmark version history with calibration results, weight assignments, and baseline results

### Requirement 24: Task Evaluation and Scoring

**User Story:** As a task author, I want to define tasks with ground truth and custom evaluators, so that the benchmark can assess diverse capabilities with appropriate scoring functions.

#### Acceptance Criteria

1. THE System SHALL support tasks with unique IDs, content, ground truth, and evaluator functions
2. THE task evaluator SHALL return a score in the range [0, 1] for any system response
3. THE System SHALL support binary scoring (0 or 1) and partial credit scoring (continuous values)
4. THE System SHALL assign tasks to categories (QA, math, coding, tool-use) for calibration coverage
5. THE System SHALL automatically assign difficulty level (Easy, Medium, Hard, Expert) based on baseline pass rates
6. THE System SHALL validate that all task IDs are unique within the task suite

### Requirement 25: Cost Curve Construction for CEI

**User Story:** As a multi-configuration system developer, I want to submit multiple runs with varying hyperparameters, so that my system's cost-quality tradeoff curve can be evaluated against the Pareto frontier.

#### Acceptance Criteria

1. THE System SHALL accept multiple runs from the same system with different hyperparameter configurations
2. WHEN building a system's cost curve, THE CEI_Calculator SHALL collect all (cost, score) points from submitted runs
3. THE System SHALL construct a non-decreasing envelope ensuring that higher cost achieves equal or higher score
4. THE System SHALL handle systems that only submit a single configuration by treating it as a single-point curve
5. THE System SHALL require at least 5 distinct cost points for a system to be eligible for CEI-based analysis, regardless of the number of submissions
6. THE System SHALL interpolate curves using linear or monotone cubic interpolation between submitted points
7. THE System SHALL require positive weights for all submitted configurations in CEI calculations
8. THE System SHALL validate task scores even when configuration weight is zero

### Requirement 26: Performance Constraints

**User Story:** As a benchmark administrator, I want the system to meet performance requirements, so that evaluations complete in reasonable time and the benchmark scales to many submissions.

#### Acceptance Criteria

1. THE Calibration_Suite SHALL complete execution for a single model in under 10 minutes
2. THE System SHALL complete evaluation of 1000 tasks in under 1 hour
3. THE Cost_Tracker SHALL introduce less than 5% overhead compared to untracked execution
4. THE Leaderboard_Generator SHALL regenerate the full leaderboard in under 1 minute with 100 system submissions
5. THE System SHALL support concurrent execution of independent task evaluations to reduce wall-clock time

### Requirement 27: Security and Data Integrity

**User Story:** As a benchmark administrator, I want to prevent tampering with logs and pricing tables, so that submission integrity is maintained and cheating is detectable.

#### Acceptance Criteria

1. THE System SHALL compute cryptographic hashes of structured logs for tamper detection
2. THE System SHALL validate that cost calculations match the declared pricing table version
3. THE System SHALL reject submissions with inconsistent cost breakdowns (C_total ≠ C_api + C_tool + C_exec + C_coord)
4. THE System SHALL validate that weighted scores match weight × score calculations
5. THE System SHALL detect and flag submissions that use undeclared models or tools
6. THE pricing table SHALL be digitally signed by benchmark administrators

### Requirement 28: Export and Reporting

**User Story:** As a researcher, I want to export benchmark data in standard formats, so that I can perform custom analysis and generate publication-quality figures.

#### Acceptance Criteria

1. THE System SHALL export leaderboard data in JSON format with all metrics and verification status
2. THE System SHALL export leaderboard data in markdown table format for documentation
3. THE System SHALL export individual run logs as JSON files following the versioned schema
4. THE System SHALL export cost breakdown summaries as CSV files for spreadsheet analysis
5. THE System SHALL export radar chart data for profile dimensions in JSON format
6. THE System SHALL export cost-quality curves for CEI analysis in CSV format

### Requirement 29: Error Handling and Edge Cases

**User Story:** As a system developer, I want the benchmark to handle failures gracefully, so that partial results are preserved and error conditions are clearly communicated.

#### Acceptance Criteria

1. WHEN a task execution times out, THE System SHALL record score = 0 and log the timeout event
2. WHEN a system crashes during execution, THE System SHALL preserve partial results and mark the run as incomplete
3. IF total weighted score Q = 0, THE System SHALL assign nWCP = +∞ and display it clearly on the leaderboard
4. IF a system produces invalid output, THE task evaluator SHALL assign score = 0 and log the validation error
5. WHEN computing division operations (WCP, CV, nWCP, etc.), THE System SHALL handle division by zero by setting appropriate defaults or assigning infinity to ensure valid computational results
6. THE System SHALL log all errors with timestamps and context for debugging

### Requirement 30: Calibration Parameter Versioning

**User Story:** As a benchmark maintainer, I want to update calibration parameters as the model ecosystem evolves, so that tier assignments remain meaningful over time.

#### Acceptance Criteria

1. THE System SHALL maintain default calibration parameters α=1.5, β=0.6, ΔA=0.1 for weak-tier and strong-tier assignment
2. WHEN calibration parameters are modified, THE System SHALL increment the benchmark version number
3. THE System SHALL archive all previous parameter sets with timestamps
4. THE System SHALL recompute model tier assignments when parameters change
5. THE System SHALL allow sensitivity analysis by executing calibration with parameter ranges
6. THE System SHALL document the rationale for parameter changes in version release notes
