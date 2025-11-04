## **Universal Problem-Solving Workflow: System Design Document**

**Version**: 1.2 (Plain Text Math Edition, Complete)
**Status**: Draft

### 1. Executive Summary

This document provides a comprehensive design for the next-generation Universal Problem-Solving Workflow (hereafter "the system"). The current system, based on a heuristic "Plan-Execute-Review" cycle, has proven effective but lacks a solid theoretical foundation, making its behavior difficult to predict and optimize.

The goal of this document is to propose a unified theoretical framework and system architecture to evolve the existing system into a computational engine that is **intelligent, robust, and optimizable**. This design integrates three core theoretical perspectives:
1.  **Cognitive Science Perspective (TPSM)**: Modeling the system as an intelligent agent that **reflects and learns**.
2.  **Computer Systems Perspective (CS/Systems)**: Formalizing the system architecture as a computational system with **verifiable properties**, centered around a **Directed Acyclic Graph (DAG)**.
3.  **Formal Methods & Optimization Perspective**: Treating the problem-solving process as a **constrained optimization problem** with **costs and probabilities**, which learns through **Counterexample-Guided Inductive Synthesis (CEGIS)**.

The final architecture will implement an intelligent **Plan-Verify-Execute-Review-Reflect (PVERR)** cycle, where each step is grounded in a solid mathematical model, enabling systematic and measurable self-improvement on complex tasks.

### 2. Unified Theoretical Model

The system's core theoretical foundation is built on the fusion of the three perspectives mentioned above, forming a unified computational model.

| **Theoretical Pillar** | **Core Metaphor** | **System Role** | **Key Contribution** |
| :--- | :--- | :--- | :--- |
| **Cognitive Agent** | The system is an intelligent agent that "thinks." | Responsible for **Intelligence & Learning** | Introduces an explicit **`Reflect`** stage for semantic learning and hypothesis generation from failures. |
| **Robust Comp. System** | The system is a rigorous "computational architecture." | Responsible for **Structure & Reliability** | Formalizes the `Plan` as a **Directed Acyclic Graph (DAG)** and defines the system's formal properties (e.g., termination, reliability). |
| **Optimal Decision Engine** | The system is a rational "decision-maker." | Responsible for **Efficiency & Optimization** | Introduces **cost/probability models** and an **executable success specification** (`success_predicate`), turning problem-solving into an optimizable decision process. |

### 3. Formal Mathematical Model

We formalize the entire workflow as an **iterative optimization process with feedback**. The goal is to find a plan that satisfies the task's success criteria within given resource constraints.

#### 3.1. Definition of Basic Elements

*   **Task**, T: A tuple `(D, C, Psi)`, where:
    *   `D`: The natural language description of the task.
    *   `C`: A set of resource constraints, e.g., `C = {max_cost, max_time}`.
    *   `Psi`: A **success predicate function**, `Psi: L -> {true, false}`, which determines if an execution log `L` satisfies the task's goal.

*   **Toolset**, E: A finite set `{Tool_1, Tool_2, ..., Tool_n}`. Each tool is a **partial function** with associated cost and failure probability models.

*   **Plan**, P: A **Directed Acyclic Graph (DAG)**, `P = (V, E)`, where:
    *   `V`: A set of nodes, where each node `v` corresponds to a tool call `(Tool_i, inputs)`.
    *   `E`: A set of edges, representing execution dependencies and data flow between nodes.

*   **Execution Log**, L: A record of the plan `P`'s execution in the environment `E`, containing the output, status, and resource consumption of each node.

*   **System State**, S_k: The state of the system at iteration `k`, containing historical information `S_k = (T, H_k)`, where `H_k = {(P_i, L_i, R_i)}` for `i` from 0 to `k-1`, is the record of past attempts.

#### 3.2. Core Iterative Function

The entire workflow is an iterative map, `Phi`, which aims to find a plan `P*` such that `Psi(Exec(P*))` is true.
> `S_{k+1} = Phi(S_k)`

This map `Phi` can be broken down into a series of functions within the PVERR cycle:
1.  **Plan**: `P_k = Planner(S_k)`. This function searches the plan space for an optimal candidate plan, aiming to maximize an expected utility function, `U(P) = E[success(P)] / E[cost(P)]`.
2.  **Verify**: `Verifier(P_k) -> {valid, invalid}`. This performs static analysis on the plan.
3.  **Exec**: `L_k = Exec(P_k)`. This executes the plan in the environment and generates a log.
4.  **Review**: `R_k = Reviewer(T, L_k) = (Psi(L_k), Counterexample(L_k))`.
5.  **Reflect**: `H_{k+1} = H_k U Reflector(P_k, L_k, R_k)`. This analyzes the failure and generates structured knowledge to update the history.

#### 3.3. Connection to Computation Theory

*   **Oracle Turing Machine**: The entire system can be viewed as a **non-deterministic Turing machine with an oracle**. The `Planner` (LLM) and `Executor` (toolset) are the oracles. The learning capability provided by the `Reflector` makes it an **adaptive Turing machine**, whose transition function adjusts based on past execution results.
*   **CEGIS (Counterexample-Guided Inductive Synthesis)**: The `Reviewer` and `Reflector` together form a CEGIS loop. The `Reviewer` acts as the Verifier, providing counterexamples, and the `Planner` acts as the Synthesizer, generating a new candidate program (Plan) based on those counterexamples.

### 4. System Architecture

The system is organized around the enhanced PVERR cycle.

#### 4.1. High-Level Architecture Diagram

```
+-----------------------------------------------------------------------------------+
|                                  Workflow Engine                                  |
|                                                                                   |
|    +------------------+      +-----------------+      +-----------------------+   |
|    |      Task        |----->|     Planner     |----->|     Plan Verifier     |   |
|    | (with Predicate) |      | (Program Synth) |      |    (Static Analysis)  |   |
|    +------------------+      +--------+--------+      +-----------+-----------+   |
|                                       ^                           |               |
|                                       |                           |               |
|    +-----------------+                |                           |               |
|    |    Reflector    |<---------------+---------------------------+               |
|    | (CEGIS Engine)  |      Feedback &                           |               |
|    +--------+--------+      Counterexamples                      | Plan (DAG)    |
|             ^                                                     |               |
|             |                                                     v               |
|    +------------------+      +-----------------+      +-----------------------+   |
|    |  Review Result   |<-----|     Reviewer    |<-----|    Execution Log      |   |
|    | (with Counterex) |      | (Formal Verifier) |      |                       |   |
|    +------------------+      +-----------------+----->|       Executor        |   |
|                                                     |  (DAG Parallel Runner)  |   |
|                                                     +-----------------------+   |
|                                                                                   |
|---------------------------------[ P V E R R   C Y C L E ]-------------------------|
+-----------------------------------------------------------------------------------+
```

#### 4.2. Core Components and Their Mathematical Correspondence

| **Core Component** | **Mathematical Model Role** | **Responsibilities & Implementation** |
| :--- | :--- | :--- |
| **Task Object** | `T = (D, C, Psi)` | Encapsulates the task description, resource constraints, and the executable success predicate. |
| **Planner** | `Planner(S_k) -> P_k` | A program synthesis engine that generates a DAG plan with cost/probability estimates, based on the current state and past reflections. |
| **Plan Verifier** | `Verifier(P_k)` | Performs static analysis on the plan, including DAG validity, type checking, and security scans. |
| **Executor** | `Exec(P_k) -> L_k` | A DAG execution engine that executes the plan according to its topological order, producing a structured execution log. |
| **Reviewer** | `Reviewer(T, L_k) -> R_k` | A formal verifier that runs the success predicate `Psi` and attempts to generate a counterexample upon failure. |
| **Reflector** | `Reflector(P_k, L_k, R_k)` | The analysis part of the CEGIS engine, performing root cause analysis and generating structured hypotheses for repair. |

### 5. Core Data Structures (Schemas)

All interactions between components are mediated by the following strictly defined Pydantic models, which are the concrete engineering implementations of the mathematical elements.

```python
from pydantic import BaseModel, Field
from typing import List, Dict, Any, Callable, Optional

# --- Task Definition (Corresponds to T) ---
class Task(BaseModel):
    task_id: str
    description: str
    success_predicate: str 
    constraints: Dict[str, Any] = Field(default_factory=dict)

# --- Plan Definition (Corresponds to P) ---
class PlanStep(BaseModel):
    step_id: str
    action: str
    inputs: Dict[str, Any]
    dependencies: List[str] = Field(default_factory=list)

class Plan(BaseModel):
    plan_id: str
    parent_plan_id: Optional[str] = None
    steps: List[PlanStep]
    estimated_cost: float = 0.0
    estimated_success_prob: float = 1.0

# --- Execution & Review (Corresponds to L and R_k) ---
class ExecutionLog(BaseModel):
    plan_id: str
    status: str 
    step_results: Dict[str, Any] 

class ReviewResult(BaseModel):
    status: str 
    feedback: str
    counterexample: Optional[Dict[str, Any]] = None

# --- Reflection & Learning (Updates history H_k) ---
class Reflection(BaseModel):
    failed_plan_id: str
    failure_category: str
    root_cause_analysis: str
    correction_hypothesis: str
```

### 6. PVERR Core Workflow Cycle

The workflow evolves from a fixed `max_loops` to a **budget-driven, state-aware PVERR cycle**, which is the concrete implementation of the iterative function `Phi`.

1.  **State 1: Planning (`q_plan`)**
    *   **Input**: `Task`, historical `Reflection` objects.
    *   **Action**: The `Planner` generates one or more candidate `Plan` objects. The system selects the optimal plan based on a utility function, such as `estimated_success_prob / estimated_cost`.
    *   **Output**: A `Plan` to be verified.

2.  **State 2: Verification (`q_verify`)**
    *   **Input**: The selected `Plan`.
    *   **Action**: The `Plan Verifier` performs static analysis. This includes checking for cycles in the dependency graph, validating tool input schemas, and scanning for potential security risks.
    *   **Output**: A validated `Plan` or an error that triggers a return to the `q_plan` state with feedback.

3.  **State 3: Execution (`q_exec`)**
    *   **Input**: A validated `Plan`.
    *   **Action**: The `Executor` executes the plan's steps, respecting the DAG dependencies. Steps without dependencies on each other can be run in parallel.
    *   **Output**: A detailed `ExecutionLog`.

4.  **State 4: Review (`q_check`)**
    *   **Input**: The original `Task` and the `ExecutionLog`.
    *   **Action**: The `Reviewer` programmatically executes the `Task.success_predicate` against the `ExecutionLog`.
    *   **Output**: A `ReviewResult`. If the status is `APPROVED`, the workflow terminates successfully. Otherwise, the status is `REVISE`.

5.  **State 5: Reflection (`q_reflect`)**
    *   **Input**: A `ReviewResult` with status `REVISE`, the failed `Plan`, and the `ExecutionLog`.
    *   **Action**: The `Reflector` performs root cause analysis on the failure, leveraging the `counterexample` if available. It generates a structured hypothesis for how to correct the plan.
    *   **Output**: A structured `Reflection` object. The system then transitions back to the `q_plan` state, using this new `Reflection` to guide the next planning cycle.

### 7. Implementation Roadmap

A phased approach is recommended to ensure a smooth transition and incremental value delivery.

#### **Phase 1: Laying the Foundation (Structural Upgrade)**
*   **Goal**: Upgrade all core interactions to use structured, verifiable data models.
*   **Tasks**:
    1.  Implement all Pydantic models defined in Section 5.
    2.  Refactor the `planner`, `executor`, and `reviewer` activities to strictly adhere to these new schemas for their inputs and outputs.
    3.  Implement the `dependencies` field within the `Plan` model. Update the `Planner`'s prompts to explicitly request dependency information when generating plans.
    4.  Refactor the `Executor` to parse the dependency graph and execute steps in the correct topological order (initially, this can still be sequential).

#### **Phase 2: Injecting Intelligence (Learning Loop)**
*   **Goal**: Implement the core failure-driven learning cycle.
*   **Tasks**:
    1.  Create a new `reflector.py` activity. Its first version will perform basic root cause analysis on the execution log.
    2.  Enhance the `reviewer.py` activity to run the `success_predicate` and, if it fails, attempt to extract a structured `counterexample` from the log.
    3.  Update the main workflow logic (`main.py`) to formally incorporate the `Reflect` stage, establishing the full PVERR cycle. The `Planner` will now receive the `Reflection` object as part of its input context.

#### **Phase 3: Achieving Robustness & Optimization (Production Grade)**
*   **Goal**: Elevate the system to production-level reliability and introduce optimization capabilities.
*   **Tasks**:
    1.  Create a `verifier.py` activity and insert this verification step between planning and execution.
    2.  Upgrade the `Executor` to support parallel execution of independent steps based on the DAG.
    3.  Replace the fixed `max_loops` with a budget-driven model based on `Task.constraints` (e.g., elapsed time, total cost).
    4.  Introduce a multi-strategy `Planner` that can select different planning algorithms (e.g., a fast template-based one vs. a slower, more thorough one) based on task complexity.

### 8. Metrics and Evaluation

To measure the system's performance and the impact of improvements, a set of evaluation metrics will be established. These metrics quantify the efficiency with which the system converges on an optimal solution.

*   **Success Rate**: The percentage of tasks that satisfy the predicate `Psi` on a standard benchmark suite.
*   **Average Cost**: The average resource consumption (time, API calls, monetary cost) to complete a task, corresponding to the model's `E[cost(P)]`.
*   **Average Iterations to Success**: The average number of PVERR cycles required to complete a task, reflecting learning efficiency.
*   **First-Pass Success Rate**: The percentage of tasks that succeed without entering the `Reflect` stage, measuring the initial accuracy of the `Planner`.
*   **Reflection Effectiveness**: The percentage increase in success rate on the next attempt after a `Reflect` cycle, measuring the performance of the learning algorithm.

---
**End of Document**