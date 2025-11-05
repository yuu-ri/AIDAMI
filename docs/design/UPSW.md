Universal Problem-Solving Workflow
---

## **Universal Problem-Solving Workflow: System Design Document**

**Version**: 1.8 (Final)
**Status**: Approved for Publication & Implementation

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

### 3. Formal Mathematical Model (Publication-Ready Formalism)

We formalize the entire workflow as an iterative dynamical system with feedback. The goal is to find a plan that satisfies the task's success criteria within given resource constraints.

#### 3.1. Formal Notation Definition

*   `T`: Represents a **Task**.
*   `P`: Represents a **Plan**.
*   `E`: Represents the **Toolset** or Environment.
*   `L`: Represents an **Execution Log**.
*   `R`: Represents the mathematical abstraction of a **Review result**.
*   `H_k`: Represents the **sequence of accumulated Reflection knowledge** at iteration `k`.
*   `Ψ` (Psi): Represents the task's success **predicate function**.
*   `cost(·)`: An overloaded **cost function**.
    *   `cost(tool, inputs): Tool × Dict → ℝ⁺`
    *   `cost(P): Plan → ℝ⁺`
*   `prob_success(·)`: Represents a **success probability function**.
*   `E[·]`: Represents the **expected value**.
*   `⊕`: Represents **sequence append**. `H ⊕ r` is equivalent to appending element `r` to sequence `H`.

#### 3.2. Definition of Basic Elements

*   **Task**, `T`: A tuple `(D, C, Ψ)`, where `D` is the description, `C` are constraints, and `Ψ` is the success predicate.
*   **Plan**, `P`: A Directed Acyclic Graph `P=(V, A)`.
*   **System State**, `S_k`: The state of the system at the beginning of iteration `k`. It is a tuple `(T, H_k)`.

#### 3.3. Core Iterative Function

The entire workflow is governed by a global state transition map `Φ`.

`Φ` is an **algorithm** with conditional branching and termination conditions, as formally defined in Section 6. It orchestrates the atomic PVERR functions to drive the system towards a solution.

*Note on Cost/Probability Models*: The cost and probability models are primarily used by the `Planner` to select efficient candidate plans. In the current model, `cost(P_k)` represents the estimated cost. Production implementations may track actual execution costs via the `ExecutionLog` for more accurate budget management.

#### 3.4. Connection to Computation Theory

*   **Oracle Turing Machine**: The entire system can be viewed as a **non-deterministic Turing machine with an oracle**. The `Planner` (LLM) and `Executor` (toolset) are the oracles. The learning capability provided by the `Reflector` makes it an **adaptive Turing machine**, whose transition function adjusts based on past execution results.
*   **CEGIS (Counterexample-Guided Inductive Synthesis)**: The PVERR loop directly implements the CEGIS paradigm:
    *   The **Synthesizer** is the `Planner`.
    *   The **Verifier** is the `Reviewer`, checking the executed plan against the specification `Ψ`.
    *   The learning loop is closed by the `Reflector`, which uses the counterexample to guide the `Synthesizer`.
    *   *Clarification*: In CEGIS terminology, the **Reviewer** plays the role of the verifier (semantic check against `Ψ`), while the system's **Plan Verifier** performs a distinct, static analysis (syntactic check) before execution.

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
| **Task Object** | `T = (D, C, Ψ)` | Encapsulates the task, constraints, and success predicate. |
| **Planner** | `Planner(context) -> P_k` | A program synthesis engine that generates a DAG plan based on the full `PlanningContext`. |
| **Plan Verifier** | `Verifier(P_k) -> V_k` | Performs static analysis on the plan. |
| **Executor** | `Exec(P_k, E) -> L_k` | Executes the plan's DAG. |
| **Reviewer** | `Reviewer(T, L_k) -> R_k` | Formal verifier against `Ψ`. |
| **Reflector** | `Reflector(...) -> reflection_k` | The CEGIS analysis engine. |

### 5. Core Data Structures (Schemas)

All interactions between components are mediated by the following strictly defined Pydantic models.

```python
from pydantic import BaseModel, Field
from typing import List, Dict, Any, Optional

# --- Task Definition (Corresponds to T) ---
class Task(BaseModel):
    task_id: str
    description: str
    success_predicate: str 
    constraints: Dict[str, Any] = Field(default_factory=dict)

# --- Formal Context for Planning ---
class PlanningContext(BaseModel):
    task: Task
    history: List['Reflection']
    verification_feedback: Optional[List[str]] = None

# --- Plan Definition (Corresponds to P) ---
class PlanStep(BaseModel):
    step_id: str
    action: str
    inputs: Dict[str, Any]
    dependencies: List[str] = Field(default_factory=list)

class Plan(BaseModel):
    plan_id: str
    steps: List[PlanStep]

# --- Verification (Corresponds to V_k) ---
class VerificationResult(BaseModel):
    status: str  # e.g., 'VALID', 'INVALID'
    issues: List[str] = Field(default_factory=list)

# --- Execution & Review (Corresponds to L and R_k) ---
class ExecutionLog(BaseModel):
    plan_id: str
    status: str
    step_results: Dict[str, Any]

class ReviewResult(BaseModel):
    status: str # e.g., 'APPROVED', 'REVISE'
    feedback: str # Provides human-readable context; not part of the core mathematical model of R_k
    counterexample: Optional[Dict[str, Any]] = None

# --- Reflection & Learning (Corresponds to reflection_k) ---
class Reflection(BaseModel):
    failed_plan_id: str
    failure_category: str
    root_cause_analysis: str
    correction_hypothesis: str
```
**Example Task Object:**
```python
task = Task(
    task_id="sales-report-q3",
    description="Generate a sales report for Q3, identifying the top 5 products by revenue.",
    success_predicate="file_exists('sales_report_q3.pdf') and is_valid_pdf('sales_report_q3.pdf')",
    constraints={
        'max_iterations': 10,
        'max_cost': 5.00,  # dollars
        'max_time': 600    # seconds
    }
)
```

### 6. PVERR Core Workflow Cycle (Formalized Transitions)

The workflow is a state-aware PVERR cycle with conditional branching. It defines the behavior of the global state transition map `Φ`.

*Note on Algorithm Signature*: While the theoretical model expresses Φ as a map from `S_k` to `S_{k+1}`, the executable algorithm is presented as a function that takes the initial `Task T` and internally manages the state sequence `S_0, S_1, ..., S_k`.

**Global State Transition Algorithm `Φ(T)`:**

The map `Φ` is defined by the following executable algorithm, which is initialized with the task `T`.

```
FUNCTION Phi(Task T):
  k = 0
  total_cost = 0
  H_k = []  // Initial empty history sequence
  verification_feedback = None

  WHILE k < T.constraints['max_iterations'] AND total_cost < T.constraints['max_cost']:
    // 1. Planning
    planning_context = PlanningContext(task=T, history=H_k, verification_feedback=verification_feedback)
    P_k = q_plan(planning_context)
    verification_feedback = None // Reset feedback after use

    // 2. Verification
    V_k = q_verify(P_k)
    IF V_k.status is 'INVALID':
      verification_feedback = V_k.issues
      CONTINUE  // Retry planning without incrementing k or cost
    
    // 3. Execution
    L_k = q_exec(P_k)
    total_cost += cost(P_k)  // Track budget post-execution (using estimated cost)
    
    // 4. Review
    R_k = q_check(T, L_k)
    IF R_k.status is 'APPROVED':
      RETURN (status: "SUCCESS", solution: P_k, history: H_k, iterations: k)
      
    // 5. Reflection & State Update
    reflection_k = q_reflect(P_k, L_k, R_k)
    H_k = H_k ⊕ reflection_k  // In-place update of history
    k += 1                     // Increment iteration counter
  
  RETURN (status: "FAILURE_BUDGET_OR_ITERATIONS", solution: None, history: H_k, iterations: k)
```

**Notes on Algorithm Behavior:**

1.  **Budget Exhaustion**: Budget exhaustion is checked at the start of each loop. Therefore, the current PVERR cycle will complete even if the execution cost of `P_k` pushes the `total_cost` over budget. This allows the system to potentially succeed on a final, slightly over-budget attempt.
2.  **Verification Loop**: The inner verification loop (a `CONTINUE` on `INVALID` status) does not have an explicit retry limit in this formal model. A production implementation should include a safeguard, such as a maximum number of verification retries per iteration or a timeout, to prevent infinite loops.

### 7. Implementation Roadmap

A phased approach is recommended to ensure a smooth transition and incremental value delivery.

#### **Phase 1: Laying the Foundation (Structural Upgrade)**
*   **Goal**: Upgrade all core interactions to use structured, verifiable data models.
*   **Tasks**: Implement schemas, refactor components to use them, handle DAG dependencies.

#### **Phase 2: Injecting Intelligence (Learning Loop)**
*   **Goal**: Implement the core failure-driven learning cycle.
*   **Tasks**: Implement `Reflector`, `Reviewer` predicate execution, and the full PVERR loop in the main workflow.

#### **Phase 3: Achieving Robustness & Optimization (Production Grade)**
*   **Goal**: Elevate the system to production-level reliability and introduce optimization capabilities.
*   **Tasks**: Implement `Verifier`, parallel execution, budget-driven termination, and a multi-strategy `Planner`.

### 8. Metrics and Evaluation

To measure the system's performance and the impact of improvements, a set of evaluation metrics will be established.

*   **Success Rate**: The percentage of tasks that satisfy the predicate `Ψ` on a standard benchmark suite.
*   **Average Cost**: The average resource consumption to complete a task.
*   **Average Iterations to Success**: The average number of PVERR cycles (`k`) required to succeed.
*   **First-Pass Success Rate**: Percentage of tasks succeeding at `k=0`.
*   **Reflection Effectiveness**: Percentage increase in success rate on the next attempt after a `Reflect` cycle.

### 9. Summary of Revisions (v1.7.1 → v1.8)

| Area | Change Implemented | Rationale |
| :--- | :--- | :--- |
| **Algorithm Behavior** | Added a "Notes" section to clarify budget overshoot and verification loop behavior. | Documents intended handling of critical edge cases for implementation. |
| **Schema Clarity** | Added a comment to the `ReviewResult.feedback` field and a concrete example for `Task.constraints`. | Improves usability and bridges the gap between formal schemas and practical application. |
| **Model vs. Implementation** | Added notes clarifying `Φ(T)` vs. `Φ(S_k)` and the use of estimated vs. actual cost. | Proactively addresses potential confusion for developers implementing the system from this specification. |

---
**End of Document**
