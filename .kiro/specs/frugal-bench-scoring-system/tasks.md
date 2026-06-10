# Implementation Plan: FrugalBench Scoring System

## Overview

This plan implements the FrugalBench scoring system in Python - a comprehensive three-layer benchmark for evaluating AI systems based on cost-efficiency. The implementation follows the design's architecture with: (1) nWCP as the primary ranking metric, (2) CEI for Pareto frontier analysis, and (3) a five-dimensional profile (P, E, O, R, S) for interpretability. The system includes automatic model calibration/tiering, adaptive difficulty weighting, cost tracking across four categories, and anti-gaming mechanisms.

## Tasks

- [ ] 1. Set up project structure and core data models
  - Create Python package structure with modules: `core`, `calibration`, `execution`, `metrics`, `leaderboard`
  - Define core data models using Pydantic: `Task`, `SystemRun`, `TaskResult`, `CostBreakdown`, `PricingTable`, `ProfileVector`
  - Implement validation rules for all data models (scores in [0,1], costs non-negative, unique IDs)
  - Set up configuration management for calibration parameters (α=1.5, β=0.6, ΔA=0.1)
  - _Requirements: 1.3, 1.4, 1.8, 2.3, 3.1-3.8, 16.1-16.8, 27.1-27.5_

- [ ] 2. Implement calibration suite and model tiering
  - [ ] 2.1 Create CalibrationSuite class with task execution
    - Implement `runCalibration(model)` method that executes calibration tasks across QA, math, coding, tool-use
    - Compute accuracy A_M, total cost C_M, and ValueScore = A_M / C_M for each model
    - Store calibration results with capability profiles
    - Ensure calibration completes in <10 minutes per model
    - _Requirements: 1.1, 1.2, 1.8, 26.1_
  
  - [ ] 2.2 Implement model tier assignment logic
    - Create `selectCheapBaseline()` method to find minimum cost model with high ValueScore
    - Implement `assignTier()` method with weak-tier criteria: cost ≤ α·C_cheap AND accuracy ≥ β·A_max
    - Implement strong-tier criteria: accuracy ≥ A_cheap + ΔA AND cost in upper 50th percentile
    - Ensure each model is assigned exactly one tier (Weak, Strong, Excluded)
    - _Requirements: 1.3, 1.4, 1.5, 1.6, 17.7_
  
  - [ ] 2.3 Add versioning and archival for calibration results
    - Implement version incrementing when calibration parameters change
    - Archive previous calibration results with timestamps
    - Store calibration results in versioned JSON format
    - _Requirements: 1.7, 23.6_

- [ ] 3. Implement difficulty weighting engine
  - [ ] 3.1 Create baseline system execution framework
    - Implement execution of baseline systems (Cheap-Only, Strong-Only, Single+Reflection, Weak-Strong Router, Multi-Agent Debate)
    - Compute pass rates p_i for each task as average success rate across baseline systems
    - Store baseline execution results for reproducibility
    - Ensure baseline execution completes within 1 hour for 1000-task suite
    - _Requirements: 2.1, 21.1-21.6_
  
  - [ ] 3.2 Implement continuous and discrete weight assignment
    - Create `assignWeights()` with continuous mode: w_i = 1 / (p_i + ε) where ε = 0.01
    - Implement discrete mode: w_i ∈ {1, 2, 4, 8} based on thresholds (0.75, 0.40, 0.15)
    - Handle edge cases: p_i = 0 (w_i = 1/ε), p_i = 1 (w_i = 1/(1+ε))
    - Ensure boundary values are assigned to higher difficulty level
    - _Requirements: 2.2, 2.3, 2.4, 2.5_
  
  - [ ] 3.3 Add weight versioning and task suite update handling
    - Version and archive weight assignments with each benchmark version
    - Block new submissions during weight recomputation after task suite updates
    - Store weights in versioned configuration files
    - _Requirements: 2.6, 2.7_

- [ ] 4. Implement comprehensive cost tracking system
  - [ ] 4.1 Create CostTracker class with four-category tracking
    - Implement `trackApiCost()` for model API calls with per-token pricing
    - Implement `trackToolCost()` for external tool/service invocations
    - Implement `trackExecutionCost()` for GPU/CPU/memory resource usage
    - Implement `trackCoordinationCost()` for meta-cognitive operations (planner, reflection, debate, verification, voting, critic)
    - _Requirements: 3.1, 3.2, 3.3, 3.4_
  
  - [ ] 4.2 Add cost aggregation and breakdown methods
    - Implement `getCostBreakdown(taskId)` returning decomposed costs per task
    - Implement `getTotalCost(runId)` aggregating costs across all tasks in run
    - Ensure C_total = C_api + C_tool + C_exec + C_coord
    - Support local model cost conversion (GPU time → equivalent cost)
    - Validate all cost values are non-negative in final computations
    - _Requirements: 3.5, 3.6, 3.7, 3.8_

- [ ] 5. Implement score aggregation with difficulty weights
  - [ ] 5.1 Create ScoreAggregator class
    - Implement `recordScore()` to store raw scores s_i ∈ [0,1] with weights
    - Implement `computeWeightedScore()` calculating Q = Σ(w_i × s_i)
    - Implement `computeDifficultyLayerScores()` partitioning by Easy, Medium, Hard, Expert
    - Support partial scoring with continuous values in [0,1]
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_
  
  - [ ] 5.2 Add failure handling and validation
    - Assign score s_i = 0 for failed/timeout/incomplete tasks
    - Handle missing tasks by excluding from both numerator and denominator
    - Validate that total difficulty weights do not sum to zero
    - _Requirements: 4.6, 4.7, 4.8_

- [ ] 6. Implement Layer 1 metric: nWCP calculator
  - [ ] 6.1 Create NWCPCalculator class
    - Implement `computeWCP()` calculating WCP = C_total / Q
    - Implement `getCheapBaseline()` retrieving baseline WCP from calibration
    - Implement `computeNWCP()` calculating nWCP = WCP_sys / WCP_cheap
    - Handle Q = 0 case by assigning nWCP = +∞
    - _Requirements: 5.1, 5.2, 5.3, 5.4_
  
  - [ ] 6.2 Add validation and interpretation logic
    - Validate all intermediate values (C_total, Q, WCP_sys, WCP_cheap) are non-negative
    - Prevent negative nWCP values through validation checks
    - Add interpretation helpers (nWCP < 1.0 = better than baseline)
    - Treat +∞ as satisfying non-negative requirement
    - _Requirements: 5.5, 5.6, 5.7, 5.8, 5.9_

- [ ] 7. Implement Layer 2 metric: CEI calculator
  - [ ] 7.1 Create cost-quality curve construction
    - Implement `buildCostQualityCurve()` creating non-decreasing envelope S̃(c)
    - Build curves ensuring higher cost achieves equal or higher score
    - Handle single-point configurations by treating as single-point curve
    - Support linear or monotone cubic interpolation between points
    - _Requirements: 6.1, 6.4, 6.6, 25.1-25.4_
  
  - [ ] 7.2 Implement Pareto frontier and integration
    - Create `buildParetoFrontier()` constructing oracle frontier Oracle(c)
    - Implement `integrateArea()` using trapezoidal rule or adaptive quadrature
    - Compute CEI numerator: ∫[S̃(c) - Cheap(c)]dc
    - Compute CEI denominator: ∫[Oracle(c) - Cheap(c)]dc
    - Calculate CEI = numerator / denominator and clamp to [0, 1]
    - _Requirements: 6.2, 6.3, 6.5, 6.6, 6.7, 6.10_
  
  - [ ] 7.3 Add CEI interpretation and validation
    - Interpret CEI = 0 as no improvement over cheap baseline
    - Interpret CEI = 1 as reaching oracle frontier
    - Require at least 5 distinct cost points for CEI-based analysis
    - Validate positive weights for all submitted configurations
    - Validate task scores even when configuration weight is zero
    - _Requirements: 6.8, 6.9, 25.5, 25.6, 25.7, 25.8_

- [ ] 8. Implement Layer 3: Profile calculator for five dimensions
  - [ ] 8.1 Implement Performance (P) dimension
    - Calculate P = Σ(w_i × s_i) / Σw_i (difficulty-weighted average accuracy)
    - Normalize to range [0, 1] where 1 = perfect weighted accuracy
    - Assign P = 0 if no tasks completed successfully
    - _Requirements: 7.1, 7.2, 7.3, 7.4_
  
  - [ ] 8.2 Implement Efficiency (E) dimension
    - Calculate E = 1 - min(nWCP, 1.0)
    - Clamp E to range [0, 1] where 1 = maximum efficiency
    - Clamp to 0% when nWCP ≥ 1.0
    - _Requirements: 8.1, 8.2, 8.3, 8.4_
  
  - [ ] 8.3 Implement Overhead (O) dimension
    - Calculate O = 1 - (C_coord / C_total)
    - Normalize to range [0, 1] where 1 = zero coordination overhead
    - Assign O = 1 by default if C_total = 0
    - Track planner, reflection, debate, verification, voting, critic separately
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5_
  
  - [ ] 8.4 Implement Robustness (R) dimension
    - Partition tasks by difficulty levels (Easy, Medium, Hard, Expert)
    - Compute per-difficulty efficiency r_d for each level
    - Calculate standard deviation σ of efficiency values
    - Compute R = 1 - σ(r_d) / τ_R where τ_R = 0.5
    - Normalize to [0, 1], clamp to 0 if σ(r_d) ≥ τ_R
    - Use normal calculation when only one difficulty level exists
    - Report error if no tasks exist
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7, 10.8, 10.9_
  
  - [ ] 8.5 Implement Stability (S) dimension
    - Collect WCP values from minimum 3 repeated runs
    - Compute coefficient of variation CV = σ(WCP) / mean(WCP)
    - Calculate S = 1 - min(CV, 1.0)
    - Normalize to [0, 1] where 1 = perfect stability
    - Clamp to 0 if CV ≥ 1.0
    - Assign S = 1.0 when CV = 0.0 (perfect stability)
    - _Requirements: 11.1, 11.2, 11.3, 11.4, 11.5, 11.6, 11.7, 11.8_

- [ ] 9. Implement execution engine with repeated runs
  - [ ] 9.1 Create ExecutionEngine class for system evaluation
    - Implement `executeRun()` orchestrating task execution with cost/score tracking
    - Implement `executeTask()` for single task execution
    - Generate unique run IDs and record seeds for reproducibility
    - Enforce 1 hour execution time limit per run
    - _Requirements: 12.3, 12.8_
  
  - [ ] 9.2 Add repeated run support
    - Implement `repeatRuns()` executing system at least 3 times with different seeds
    - Record seed value for each run
    - Compute mean and standard deviation of WCP across runs
    - Continue with remaining runs when execution fails/times out
    - _Requirements: 12.1, 12.2, 12.6, 12.7_
  
  - [ ] 9.3 Implement structured JSON logging
    - Generate JSON log entry per task with run_id, seed, repeat_run_idx, system_name, system_type, task_id
    - Include score, weighted_score, difficulty, weight, baseline_pass_rate
    - Include cost breakdown: api_cost, tool_cost, execution_cost, coordination_cost, total_cost
    - Include usage metrics: input_tokens, output_tokens, model_call_count, tool_call_count
    - Include model_assignment for Weak-Strong systems (weak_model, strong_model)
    - Include coordination_steps array with operation types and costs
    - Include execution_time_ms, final_answer, grading_result
    - Follow versioned schema for forward compatibility
    - _Requirements: 13.1, 13.2, 13.3, 13.4, 13.5, 13.6, 13.7, 13.8_
  
  - [ ] 9.4 Add system type classification
    - Classify systems as Single, Weak-Strong, Multi-Agent, Router-Based, or Hybrid
    - Validate Weak-Strong systems specify models from calibrated weak/strong pools
    - Support router-based systems with multiple model tracking
    - _Requirements: 17.1-17.8, 22.1-22.6_

- [ ] 10. Checkpoint - Ensure core metrics work end-to-end
  - Execute a simple test system through calibration, execution, and metric calculation
  - Verify nWCP, CEI, and profile (P, E, O, R, S) all compute correctly
  - Ensure all tests pass, ask the user if questions arise

- [ ] 11. Implement leaderboard generation system
  - [ ] 11.1 Create LeaderboardGenerator class
    - Implement `rankByNWCP()` sorting systems by nWCP ascending (lower is better)
    - Implement `formatEntry()` creating leaderboard entries with all metrics
    - Display system_name, system_type, nWCP, CEI, and profile (P, E, O, R, S)
    - Include generation timestamp and benchmark version number
    - _Requirements: 14.1, 14.2, 14.8_
  
  - [ ] 11.2 Add verification status management
    - Assign verification_status = "self-reported" on first submission
    - Support status update to "reproduced" after independent reproduction with proof
    - Support status update to "verified" after maintainer re-execution with confirmation
    - Display icons: ❌ self-reported, ⚠️ reproduced, ✅ verified
    - _Requirements: 14.3, 15.1, 15.2, 15.3, 15.4, 15.5, 15.6, 15.7_
  
  - [ ] 11.3 Add filtering and export capabilities
    - Implement filtering by system_type
    - Implement filtering by verification_status
    - Use CEI as tiebreaker when nWCP values are identical
    - Export leaderboard in JSON and markdown formats
    - _Requirements: 14.4, 14.5, 14.6, 14.7_

- [ ] 12. Implement anti-gaming mechanisms
  - [ ] 12.1 Add robustness-based gaming detection
    - Compute per-difficulty-level efficiency for Easy, Medium, Hard, Expert
    - Assign low R score when high variance across difficulty levels
    - Display R dimension prominently on leaderboard
    - Generate radar charts showing all five profile dimensions
    - Flag systems with nWCP < 1 AND R < 0.5 showing selective performance patterns
    - _Requirements: 18.1, 18.2, 18.3, 18.4, 18.5_
  
  - [ ] 12.2 Add coordination overhead tracking
    - Track all planner, reflection, debate, verification, voting, critic operations
    - Assign O < 0.7 when coordination cost exceeds 30% of total cost
    - Support analysis of coordination cost vs performance gain
    - Reflect increased coordination without proportional performance in decreased O and E scores
    - _Requirements: 19.1, 19.2, 19.3, 19.4, 19.5_
  
  - [ ] 12.3 Enforce mandatory stability reporting
    - Reject leaderboard submissions with fewer than 3 repeated runs
    - Report mean nWCP, standard deviation, and coefficient of variation
    - Assign S < 0.7 when CV > 0.3
    - Display S dimension prominently
    - Support filtering for systems with S > threshold for production-readiness
    - _Requirements: 20.1, 20.2, 20.3, 20.4, 20.5_

- [ ] 13. Implement pricing table management
  - [ ] 13.1 Create PricingTable data model
    - Store per-token costs with input_cost_per_token, output_cost_per_token, minimum_cost
    - Store tool costs with per-call or per-unit pricing
    - Store execution costs: GPU cost per hour by type, CPU cost per hour, memory cost per GB-hour
    - Validate all pricing values are non-negative
    - _Requirements: 16.1, 16.2, 16.3, 16.4, 16.8_
  
  - [ ] 13.2 Add versioning and historical reproducibility
    - Increment version number when pricing table updated
    - Archive previous versions with timestamps
    - Associate each evaluation run with pricing table version
    - Support cost recalculation using archived pricing tables
    - Store config_hash for drift detection
    - _Requirements: 16.5, 16.6, 16.7, 23.1, 23.2, 23.3, 23.4, 23.5_

- [ ] 14. Implement task evaluation and scoring
  - [ ] 14.1 Create Task data model with validation
    - Support unique task IDs, content, ground truth, evaluator functions
    - Return scores in range [0, 1] from evaluators
    - Support binary scoring (0 or 1) and partial credit (continuous)
    - Assign tasks to categories (QA, math, coding, tool-use)
    - Validate all task IDs are unique within task suite
    - _Requirements: 24.1, 24.2, 24.3, 24.4, 24.6_
  
  - [ ] 14.2 Add automatic difficulty assignment
    - Automatically assign difficulty level (Easy, Medium, Hard, Expert) based on baseline pass rates
    - Store baseline pass rate with each task
    - _Requirements: 24.5_

- [ ] 15. Add error handling and edge cases
  - [ ] 15.1 Handle zero weighted score scenario
    - Set WCP = +∞, nWCP = +∞ when Q = 0
    - Set E = 0, P = 0
    - Log warning about system failing all tasks
    - Allow system on leaderboard (ranked last) with other computable dimensions
    - _Requirements: 5.4, 5.8_
  
  - [ ] 15.2 Handle missing baseline and insufficient runs
    - Raise MissingBaselineError when cheap baseline not found
    - Block leaderboard generation without baseline
    - Handle fewer than 3 runs by setting S = null and verification = "self-reported"
    - _Requirements: 12.1, 11.1_
  
  - [ ] 15.3 Handle cost validation and oracle frontier issues
    - Raise InvalidCostError on negative cost values
    - Set CEI = 0 when oracle frontier provides no improvement over cheap baseline
    - Log warnings for anomalous situations
    - _Requirements: 3.8_
  
  - [ ] 15.4 Handle weak-strong model assignment and weight issues
    - Raise InvalidSystemConfigError when Weak-Strong system missing model_assignment
    - Validate models are from correct calibrated tiers
    - Apply smoothing when pass rate = 0 to prevent infinite weights
    - _Requirements: 2.4, 22.4_

- [ ] 16. Add performance optimizations
  - [ ] 16.1 Optimize calibration and execution
    - Implement caching for calibration results per model version
    - Support parallel task execution within runs
    - Stream results to disk instead of holding all in memory
    - Ensure <5% overhead from cost tracking
    - _Requirements: 26.1, 26.2, 26.3_
  
  - [ ] 16.2 Optimize CEI and leaderboard generation
    - Cache oracle frontier between system additions
    - Pre-compute metrics during evaluation
    - Support incremental leaderboard updates
    - Ensure <1 minute regeneration with 100 submissions
    - Ensure <1 second CEI calculation per system
    - _Requirements: 26.4_

- [ ] 17. Final checkpoint and integration testing
  - Run full end-to-end evaluation with multiple system types (Single, Weak-Strong, Multi-Agent)
  - Verify leaderboard generation with all three layers (nWCP, CEI, profile)
  - Test anti-gaming mechanisms with synthetic gaming attempts
  - Ensure pricing table versioning and reproducibility work correctly
  - Verify structured logging produces valid JSON with all required fields
  - Ensure all tests pass, ask the user if questions arise

## Notes

- Tasks build incrementally, establishing data models before computation logic
- Each task references specific requirements for traceability
- Checkpoints ensure validation at reasonable breaks
- Core metrics (nWCP, CEI, profile) are implemented before leaderboard and anti-gaming features
- Anti-gaming mechanisms rely on robustness, overhead, and stability dimensions
- Performance optimizations are added after core functionality is working

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1"] },
    { "id": 1, "tasks": ["2.1", "14.1"] },
    { "id": 2, "tasks": ["2.2", "3.1", "4.1", "14.2"] },
    { "id": 3, "tasks": ["2.3", "3.2", "4.2", "5.1"] },
    { "id": 4, "tasks": ["3.3", "5.2", "6.1", "13.1"] },
    { "id": 5, "tasks": ["6.2", "7.1", "8.1", "13.2"] },
    { "id": 6, "tasks": ["7.2", "8.2", "8.3", "9.1"] },
    { "id": 7, "tasks": ["7.3", "8.4", "9.2"] },
    { "id": 8, "tasks": ["8.5", "9.3"] },
    { "id": 9, "tasks": ["9.4"] },
    { "id": 10, "tasks": ["11.1"] },
    { "id": 11, "tasks": ["11.2", "11.3", "12.1"] },
    { "id": 12, "tasks": ["12.2", "12.3", "15.1"] },
    { "id": 13, "tasks": ["15.2", "15.3", "15.4", "16.1"] },
    { "id": 14, "tasks": ["16.2"] }
  ]
}
```
