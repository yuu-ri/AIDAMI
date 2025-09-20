# Part 1: Architecture

## High-Level Architecture Overview

The "Intelligent Office" is a **Workflow Orchestrator** managing specialized AI agents with task decomposition, robust scalability, advanced observability, and a plugin-based tool system. The system processes user requests asynchronously, decomposes complex tasks, validates plans, executes them, and learns from both successes and failures.

```mermaid
%% Mermaid version: 10.9.4 %%
graph TD
    U[User/Client Application] --> G[API Gateway/Receptionist]
    G --> TD[Task Decomposition Service/Analyst]
    TD --> O[Orchestration Service/Office Manager]
    
    O --> P[Planner Service/Strategist]
    O --> PV[Plan Validator Service/Senior Reviewer]
    O --> E[Executor Service/Worker Bee]
    O --> R[Reviewer Service/QA]
    O --> TB[Tool Broker Service/IT Security]
    O --> DLQ[Dead Letter Queue/Task Morgue]
    
    P --> V[Vector DB/Archives]
    P --> L[LLM Abstraction Service/Model Selector]
    PV --> L
    R --> L
    TB --> L
    
    E --> TR[Tool Registry & Runtime/Workshop]
    TR --> XS[External Systems/APIs]
    TB --> TR
    
    O --> S[State & Log Storage/Archives]
    O --> M[Monitoring & Observability/Control Room]
    
    G --> UF[User Feedback Store/Suggestion Box]
    UF --> V
    
    subgraph "The Intelligent Office"
        TD
        O
        P
        PV
        E
        R
        TB
        TR
        L
        S
        V
        UF
        M
        DLQ
    end
```

## Detailed Component Breakdown

### 1. API Gateway/Receptionist (The "Receptionist")
- **Purpose**: Single entry point for user requests, handling authentication and routing
- **Responsibilities**:
  - Authenticate and authorize users
  - Validate and route requests to Task Decomposition Service
  - **Asynchronous Handling**: Return `task_id` with `202 Accepted` for long-running tasks, support **webhooks** for completion notifications
  - **Rate Limiting & Quotas**: Enforce dynamic limits to prevent LLM overuse
  - **Idempotency**: Check for duplicate `task_id`s to prevent redundant processing
  - Expose `/tasks/{task_id}/status` endpoint for progress updates
  - Log requests to Monitoring & Observability Service

### 2. Task Decomposition Service/Analyst (The "Task Analyst")
- **Purpose**: Breaks down complex user tasks into smaller, manageable subtasks
- **Responsibilities**:
  - Receive tasks from API Gateway
  - Use LLM (via LLM Abstraction Service) to split tasks into subtasks with dependencies
  - Create structured task DAG with resource/tool hints
  - Pass structured subtasks to Orchestrator

### 3. Orchestration Service/Office Manager (The "Project Manager")
- **Purpose**: Coordinates workflow across subtasks, managing state and iterations
- **Responsibilities**:
  - Receive subtasks from Task Decomposition Service
  - **Task Prioritization**: Use **message queue** (e.g., RabbitMQ) for priority handling
  - Maintain Task Scratchpad with task state, plans, results, and feedback
  - Coordinate between Planner, Plan Validator, Executor, Reviewer, and Tool Broker
  - **Model Selection**: Request appropriate LLM tiers via LLM Abstraction Service
  - **Circuit Breakers**: Handle downstream failures (e.g., LLM rate limits)
  - Send failed tasks to **Dead Letter Queue** for inspection
  - Store successful/failed plans and user feedback in Vector DB
  - Send metrics and traces to Monitoring & Observability Service

### 4. Planner Service/Strategist (The "Strategist")
- **Purpose**: Generates actionable plans for subtasks, learning from past experiences
- **Responsibilities**:
  - Query Vector DB for similar successful and failed plans
  - **Context Optimization**: Truncate irrelevant Task Scratchpad data to reduce token usage
  - Use appropriate LLM tier (via LLM Abstraction Service) to generate plans with **confidence scores**
  - **Dynamic Adjustment**: Adjust plans mid-execution based on Executor feedback
  - Output structured plans following JSON schema contract

### 5. Plan Validator Service/Senior Reviewer (The "Senior Reviewer")
- **Purpose**: Performs pre-execution validation on plans
- **Responsibilities**:
  - Validate plan syntax and structure using **rule-based checks** and fast models
  - Check for logical flaws (e.g., incorrect tool parameters, security violations)
  - **Timeout Mechanism**: Limit validation duration
  - Enforce JSON schema compliance
  - Return "APPROVED" or "REJECTED with feedback" to Orchestrator

### 6. Tool Broker Service/IT Security (The "IT Security & Procurement")
- **Purpose**: Manages and secures tool usage with approval workflows
- **Responsibilities**:
  - Receive and evaluate tool requests from plans
  - **AI Gatekeeper**: Use LLM to assess tool safety and appropriateness
  - Present tools for human approval based on risk classification
  - **Audit Trails**: Log all approvals and executions
  - Update Tool Registry with approved tools and **dynamic sandboxing policies**

### 7. Tool Registry & Runtime/Workshop (The "Workshop")
- **Purpose**: Stores and executes tools securely
- **Responsibilities**:
  - Maintain tool definitions, metadata, and security policies
  - **Plugin System**: Support standardized API for adding new tools
  - Execute tools in **sandboxed environment** (containers/Lambda) with runtime monitoring
  - Send execution metrics to Monitoring & Observability Service

### 8. Executor Service/Worker Bee (The "Worker Bee")
- **Purpose**: Executes validated plans step by step
- **Responsibilities**:
  - Iterate through plan steps, invoking tools via Tool Registry
  - **Parallel Execution**: Run independent steps concurrently where possible
  - **Retries**: Use exponential backoff for transient failures
  - Maintain **idempotency** for all operations
  - Capture and return comprehensive logs, outputs, and errors

### 9. Reviewer Service/QA (The "Quality Assurance")
- **Purpose**: Evaluates execution results against objectives
- **Responsibilities**:
  - Use appropriate LLM tier to assess results against task objectives and user-defined criteria
  - Provide **structured feedback** with clear status (APPROVE/REVISE/ESCALATE)
  - **Escalation Path**: Allow human override for critical tasks
  - Support objective evaluation metrics

### 10. LLM Abstraction Service/Model Selector (The "Model Selector")
- **Purpose**: Manages LLM interactions and model tier selection
- **Responsibilities**:
  - **Model Policy Engine**: Route requests to appropriate model tiers based on task type and cost
  - Handle different model types:
    - **Encoders** (SBERT, MiniLM): embeddings, similarity search
    - **Small LLMs** (GPT-4o-mini): validation, quick tasks
    - **Medium LLMs** (GPT-4o): planning, decomposition
    - **Large LLMs** (capable models): strategic decisions, reviews
  - **Cost Management**: Token tracking, budget enforcement, caching
  - **Fallback Logic**: Handle rate limits and failures gracefully

### 11. Vector DB/Archives (The "Corporate Knowledge")
- **Purpose**: Stores learned knowledge and retrieval for context
- **Responsibilities**:
  - Store successful plans with embeddings for similarity search
  - Store failed plans with failure reasons and lessons learned
  - Store user feedback and annotations
  - Support semantic search for plan retrieval
  - Use **Sentence-BERT embeddings** for plan similarity

### 12. State & Log Storage/Archives (The "Archives")
- **Purpose**: Persistent storage for task states and execution logs
- **Responsibilities**:
  - **Structured Data**: Use PostgreSQL for task states, plan history
  - **Transient State**: Use Redis for active task scratchpads
  - **Data Partitioning**: Optimize for query patterns and retention
  - **Retention Policies**: Archive old data to manage costs

### 13. User Feedback Store/Suggestion Box (The "Suggestion Box")
- **Purpose**: Captures user feedback for continuous improvement
- **Responsibilities**:
  - Provide **Feedback API** for structured user input
  - Store feedback with task/plan associations
  - Feed feedback into Vector DB for future plan improvement

### 14. Monitoring & Observability/Control Room (The "Control Room")
- **Purpose**: Comprehensive system monitoring and health tracking
- **Responsibilities**:
  - **Metrics Collection**: Task completion rates, LLM token usage, cost tracking using **Prometheus/Grafana**
  - **Distributed Tracing**: End-to-end request tracking with **Jaeger/OpenTelemetry**
  - **Log Aggregation**: Centralized logging with **ELK Stack/Datadog**
  - **SLO Monitoring**: Track service level objectives and alert on violations
  - **Cost Telemetry**: Per-tenant spend tracking and budget alerts

### 15. Dead Letter Queue/Task Morgue (The "Task Morgue")
- **Purpose**: Handles unrecoverable failed tasks
- **Responsibilities**:
  - Receive tasks that fail after max iterations or encounter unrecoverable errors
  - Support manual inspection and analysis
  - Enable reprocessing after fixes
  - Maintain failure analytics for system improvement

## Revised Workflow Sequence

```mermaid
%% Mermaid version: 10.9.4 %%
sequenceDiagram
    participant U as User
    participant G as API Gateway
    participant TD as Task Decomposition
    participant O as Orchestrator
    participant P as Planner
    participant V as Vector DB
    participant PV as Plan Validator
    participant TB as Tool Broker
    participant E as Executor
    participant TR as Tool Registry
    participant R as Reviewer
    participant L as LLM Abstraction
    participant S as State Storage
    participant M as Monitoring
    participant DLQ as Dead Letter Queue

    U->>G: Submit Task Request
    G->>M: Log Request Metrics
    G->>TD: Forward Task (check idempotency)
    TD->>L: Request Task Decomposition
    L-->>TD: Structured Subtask DAG
    TD->>O: Send Subtasks + Dependencies
    O->>S: Initialize Task State

    loop For Each Subtask Until Complete or Max Iterations
        O->>P: Request Plan Generation
        P->>V: Query Similar Plans (success/failure)
        V-->>P: Historical Examples
        P->>L: Generate Plan (appropriate model tier)
        L-->>P: Generated Plan + Confidence
        P-->>O: Proposed Plan

        O->>PV: Validate Plan
        PV->>L: Schema + Rule Validation (fast model)
        L-->>PV: Validation Results
        PV-->>O: APPROVED/REJECTED + Feedback

        alt Plan Approved
            O->>TB: Request Tool Approvals
            TB->>L: AI Safety Assessment
            L-->>TB: Safety Evaluation
            TB-->>O: Tool Authorization + Policies
            
            O->>E: Execute Plan
            E->>TR: Execute Tools (with sandbox policies)
            TR-->>E: Tool Results + Logs
            E-->>O: Execution Results
            O->>S: Save Execution State

            O->>R: Review Results
            R->>L: Quality Assessment (appropriate model)
            L-->>R: Quality Evaluation
            R-->>O: Structured Feedback (APPROVE/REVISE/ESCALATE)

            alt Review Approved
                O->>S: Mark Subtask Complete
                O->>V: Store Successful Plan
                O->>M: Log Success Metrics
                note right of O: Subtask Complete
            else Review Requires Revision
                O->>S: Update Scratchpad with Feedback
                O->>M: Log Revision Required
            end
        else Plan Rejected
            O->>S: Update Scratchpad with Validation Issues
            O->>M: Log Validation Failure
        end
    end

    alt All Subtasks Complete
        O->>G: Task Completed Successfully
        G->>U: Notify Completion (webhook/poll)
        O->>M: Log Task Success
    else Max Iterations Exceeded or Fatal Error
        O->>DLQ: Send to Dead Letter Queue
        O->>G: Task Failed
        G->>U: Notify Failure + Next Steps
        O->>M: Log Task Failure
    end

    U->>G: Check Task Status (optional)
    G->>S: Query Task State
    S-->>G: Current Status + Progress
    G-->>U: Status Response
```

## Key Architectural Principles

### 1. **Modularity & Loose Coupling**
- Services are stateless and containerized (Docker/Kubernetes)
- Communication via **message queues** (RabbitMQ, AWS SQS) for decoupling
- Well-defined API contracts between services

### 2. **Scalability**
- **Auto-scaling** and **load balancing** (AWS ALB) handle traffic spikes
- **Priority queues** for task prioritization
- Horizontal scaling of stateless services

### 3. **Observability**
- **Comprehensive Monitoring**: Prometheus/Grafana for metrics
- **Distributed Tracing**: Jaeger/OpenTelemetry for request tracking
- **Centralized Logging**: ELK Stack/Datadog for log aggregation
- **SLO Tracking**: Service level objectives with alerting

### 4. **Security**
- **End-to-end encryption** for data in transit and at rest
- **Strict sandboxing** (Docker containers, AWS Lambda) for tool execution
- **Runtime monitoring** and **audit trails** for compliance
- **PII detection and redaction** before LLM processing

### 5. **Cost Management**
- **Model tiering** with appropriate models for different tasks
- **Batch processing** and **context optimization** to reduce token usage
- **Intelligent caching** (Redis with TTL) for repeated queries
- **Per-tenant cost tracking** and budget enforcement

### 6. **Reliability**
- **Circuit breakers** for handling downstream failures
- **Retries with exponential backoff** and jitter
- **Idempotency keys** for all operations with side effects
- **Dead letter queues** for unrecoverable failures
- **Timeout mechanisms** to prevent resource leaks

### 7. **Learning & Feedback**
- **Vector DB storage** of successful and failed plans with embeddings
- **User feedback integration** for continuous improvement
- **Failure analysis** and pattern recognition
- **Plan optimization** based on historical performance

### 8. **Extensibility**
- **Plugin system** for easy tool integration
- **Multi-modal support** (text, images, documents)
- **Flexible model integration** via LLM Abstraction Service
- **Configurable policies** for different use cases

## Plan Contract Schema

All plans must conform to this JSON schema for consistency between Planner and Executor:

```json
{
  "$id": "https://example.com/schemas/ai-plan.json",
  "title": "AI Task Plan",
  "type": "object",
  "required": ["plan_id", "task_id", "steps", "metadata"],
  "properties": {
    "plan_id": { "type": "string" },
    "task_id": { "type": "string" },
    "generated_at": { "type": "string", "format": "date-time" },
    "confidence": { "type": "number", "minimum": 0, "maximum": 1 },
    "metadata": {
      "type": "object",
      "properties": {
        "priority": { "type": "integer", "minimum": 0 },
        "required_tools": { "type": "array", "items": { "type": "string" } },
        "estimated_cost_tokens": { "type": "number" },
        "estimated_duration_seconds": { "type": "number" }
      }
    },
    "steps": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["step_id", "action", "inputs", "idempotency_key"],
        "properties": {
          "step_id": { "type": "string" },
          "action": { "type": "string" },
          "inputs": { "type": "object" },
          "outputs_expectation": { "type": ["object", "null"] },
          "timeout_seconds": { "type": "integer", "default": 300 },
          "parallelizable": { "type": "boolean", "default": false },
          "idempotency_key": { "type": "string" },
          "dependencies": { "type": "array", "items": { "type": "string" } }
        }
      }
    }
  }
}
```

## Implementation Technology Stack

### Core Infrastructure
- **API Gateway**: FastAPI or AWS API Gateway
- **Orchestration**: Temporal or Apache Airflow
- **Message Queues**: RabbitMQ or AWS SQS
- **Load Balancing**: AWS ALB or NGINX

### Data Storage
- **Structured Data**: PostgreSQL for task states and metadata
- **Transient State**: Redis for active sessions and caching
- **Vector Storage**: Pinecone or Chroma for plan embeddings
- **Object Storage**: AWS S3 for logs and artifacts

### AI/ML Services
- **Model Abstraction**: LangChain or custom wrapper
- **Embeddings**: Sentence-BERT, MiniLM
- **Small LLMs**: GPT-4o-mini, Llama-2 7B
- **Large LLMs**: GPT-4o, Claude, or Grok 3

### Observability
- **Metrics**: Prometheus + Grafana
- **Tracing**: Jaeger or OpenTelemetry
- **Logging**: ELK Stack or Datadog
- **Alerting**: PagerDuty or AWS CloudWatch

### Security & Runtime
- **Containerization**: Docker + Kubernetes
- **Sandboxing**: Docker containers, AWS Lambda
- **Secrets Management**: HashiCorp Vault or AWS KMS
- **Identity**: Auth0 or AWS Cognito

## Success Metrics & SLOs

### Key Performance Indicators
- **Task Success Rate**: ≥99% for non-transient errors
- **Median Task Completion**: <10s for simple tasks, <60s for complex tasks
- **System Availability**: ≥99.9% uptime
- **Mean Time to Detection**: <1 minute for failures
- **Cost Efficiency**: Token cost per successful task optimization

### Monitoring Dashboards
- Task throughput and success rates by type
- LLM token usage and costs by model tier
- Queue depths and processing latencies
- Error rates and failure classifications
- User satisfaction and feedback trends

---

## Framework Mapping Analysis

Your **"Intelligent Office" AI App Architecture** is very strong — it actually aligns quite well with modern **advanced AI app frameworks** (e.g., LangGraph, Haystack Orchestrators, CrewAI, and OpenAI’s recommended AI Agent patterns) and goes a step further by incorporating enterprise-grade scalability, observability, and governance.

### Where Your Architecture Aligns with Advanced AI App Frameworks

#### 1. **Workflow Orchestration as a First-Class Citizen**
* **Best Practice:** Use an orchestrator (e.g., Temporal, Prefect, LangGraph) for managing complex, stateful workflows.
* **Your Design:** Orchestration Service acts as the "Project Manager," coordinating planning, validation, execution, retries, and state persistence — exactly what modern AI orchestration frameworks recommend.

#### 2. **Task Decomposition + Planning Loop**
* **Best Practice:** Break complex tasks into subtasks, plan iteratively, validate before execution.
* **Your Design:** Task Decomposition Service + Planner + Plan Validator + Reviewer create a feedback loop that mirrors **Chain-of-Thought + Critic + ReAct-style loops** used in advanced agentic frameworks.

#### 3. **Model Abstraction & Cost-Aware Model Selection**
* **Best Practice:** Route tasks to the right LLM tier based on complexity and budget (avoid "just throw everything at GPT-4").
* **Your Design:** LLM Abstraction Service chooses between small, medium, and large models, with caching, budget enforcement, and fallback logic — this is exactly how OpenAI and Anthropic recommend building **cost-efficient AI apps**.

#### 4. **Tool-Oriented Execution**
* **Best Practice:** Treat external APIs, functions, and databases as tools with secure, auditable usage.
* **Your Design:** Tool Broker + Tool Registry + sandboxed execution runtime = enterprise-grade **tool calling framework** with approval workflows.

#### 5. **Learning from Past Experience**
* **Best Practice:** Store plans, successes, and failures in a vector store to improve future planning.
* **Your Design:** Vector DB ("Corporate Knowledge") contains successes, failures, and user feedback. This is an **experience replay mechanism**, key to building self-improving systems.

#### 6. **Observability & Reliability**
* **Best Practice:** Include tracing, metrics, logging, retries, circuit breakers, and DLQs for production reliability.
* **Your Design:** Comprehensive Monitoring & Observability Service + Dead Letter Queue + idempotency + exponential backoff — this is **SRE-grade thinking**, rarely seen in early-stage AI apps.

#### 7. **Security & Governance**
* **Best Practice:** Sandbox untrusted code, audit tool use, enforce approval workflows for risky actions.
* **Your Design:** Tool Broker acts as a gatekeeper with human-in-the-loop approval, plus sandbox execution — excellent for enterprise and regulated industries.

### Where Your Design Goes Beyond Typical AI Frameworks

* **Explicit Plan Schema**: Your JSON schema contract for plans is stricter than what most frameworks enforce. This ensures reliability, easier debugging, and deterministic execution.
* **Circuit Breakers & Quotas**: Not always implemented in early AI apps but critical for production-grade systems.
* **Cost Telemetry & Per-Tenant Budgeting**: This is a big step toward **multi-tenant SaaS readiness**.
* **Human Escalation Paths**: Most frameworks leave human-in-the-loop to the implementer; you've made it a core part of the architecture.
* **Dead Letter Queue**: Many AI orchestration frameworks don't handle permanent failure gracefully — you do.

## Direct Solution Mapping Table

| **Component in Your AI App Architecture**                         | **Existing Open-Source Project / Framework**                                                                                                       | **How to Use It (and What You Still Need to Build)**                                                                                                                                                                                                                                                                                    |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Orchestrator (Planner → Validator → Executor → Reviewer loop)** | **Temporal** ([temporalio/temporal](https://github.com/temporalio/temporal))                                                                       | Use Temporal workflows to implement your orchestration logic — each stage (plan, validate, execute, review) becomes a workflow step or activity. Temporal gives retries, backoff, state persistence, DLQ routing, and visibility. You still need to write your planner/validator/QA logic as code that runs inside Temporal activities. |
| **Task Decomposer / Planner**                                     | **Orion** ([AshishKumar4/Orion](https://github.com/AshishKumar4/Orion)) or **AutoGen** ([microsoft/autogen](https://github.com/microsoft/autogen)) | Use Orion or AutoGen to define multi-agent planner agents that break user requests into subtasks. You may need to customize to match your domain and integrate with Temporal to persist plan steps and results.                                                                                                                         |
| **Plan Validator**                                                | No strong open-source match                                                                                                                        | You likely need to build this from scratch — implement schema validation, tool parameter checking, business-rule verification. You can leverage **Pydantic** (Python) for schema validation, but the validator logic itself will be custom.                                                                                             |
| **Tool Broker & Tool Registry**                                   | **llm-tools-hub** or **LangChain Tool API**                                                                                                        | Use llm-tools-hub to register and expose tools to agents. Extend it with access control policies, logging, and a sandbox (e.g. run tools in isolated containers or subprocesses).                                                                                                                                                       |
| **Sandboxed Tool Runtime**                                        | No direct solution — can adapt                                                                                                                     | Use Docker or Firecracker microVMs to sandbox untrusted tool execution. You'll need to build orchestration + security layer yourself (policy enforcement, network isolation).                                                                                                                                                           |
| **Execution Engine (Agents calling tools)**                       | **ModelScope-Agent** or **LangChain**                                                                                                              | Use these frameworks to manage tool invocation, handle inputs/outputs, and LLM calls. Connect execution logs back to Temporal so you maintain state/history.                                                                                                                                                                            |
| **Vector DB for Memory & Feedback Loop**                          | **Weaviate**, **Qdrant**, or **Milvus**                                                                                                            | Use these open-source vector databases to store successful plans, failed attempts, scratchpads. Build a feedback service that learns from stored data and improves future planning.                                                                                                                                                     |
| **Observability & Monitoring**                                    | **OpenTelemetry + Temporal Web UI**                                                                                                                | Use Temporal's built-in Web UI for workflow history and integrate OpenTelemetry for tracing. You may need to build dashboards (Grafana/Prometheus) for LLM cost monitoring and tool usage analytics.                                                                                                                                    |
| **Dead Letter Queue (DLQ) & Retry Management**                    | **Temporal (native)**                                                                                                                              | Temporal gives you retry policies, DLQ workflows, and failure history. Minimal code needed here beyond configuring retry policies and building handlers for DLQ events.                                                                                                                                                                 |
| **Cost Telemetry & Billing**                                      | No open-source drop-in                                                                                                                             | Build a custom middleware to wrap each LLM/tool call, record token usage/cost, and export to Prometheus or your monitoring system.                                                                                                                                                                                                      |
| **Human-in-the-Loop Approval**                                    | **AutoGen (Human Proxy Agent)**                                                                                                                    | Use AutoGen's human proxy agents or build custom Temporal "signal" steps to wait for human input before proceeding.                                                                                                                                                                                                                     |

## Framework-Specific Mapping

### 1. Mapping to **LangGraph** (LangChain's Agentic Workflow Framework)

| **Your Component**             | **LangGraph Equivalent**                                                       | **Implementation Notes**                                              |
| ------------------------------ | ------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| **API Gateway/Receptionist**   | External FastAPI endpoint + LangServe for async task submission                | LangServe can expose `/tasks/{task_id}` endpoints & handle streaming. |
| **Task Decomposition Service** | `GraphNode` with LLMChain (task decomposition prompt)                          | DAG node that outputs subtask graph (edges = dependencies).           |
| **Orchestrator**               | **LangGraph Controller** (root node) + `MemorySaver` for state persistence     | LangGraph supports scratchpad state across steps.                     |
| **Planner Service**            | `GraphNode` with LLMChain (planning prompt) + VectorStoreRetriever for context | Retrieve past plans, feed into prompt, output structured plan.        |
| **Plan Validator**             | `Tool` or `GraphNode` with fast LLM / rule-based validation                    | Reject plan by raising control signal → Orchestrator retries.         |
| **Executor Service**           | `ToolExecutor` (LangChain Tool layer)                                          | Can run tools sequentially or in parallel.                            |
| **Tool Broker & Registry**     | LangChain's `Tool` interface + custom gating middleware                        | Add custom approval policy before registering a tool.                 |
| **Reviewer Service**           | Separate `GraphNode` running evaluation LLM                                    | Could branch: if REVISE, loop back to Planner node.                   |
| **LLM Abstraction Service**    | `ChatOpenAI` with custom router + callback manager                             | Implement model tier routing using custom LLM wrapper.                |
| **Vector DB**                  | `VectorStore` (e.g. Chroma, Pinecone)                                          | Use `SimilaritySearchRetriever` to fetch similar past plans.          |
| **State & Logs**               | LangGraph internal state store + Redis backend                                 | Use `MemorySaver` or `CheckpointSaver` for resumable workflows.       |
| **Monitoring & Observability** | LangSmith (LangChain tracing & analytics)                                      | Gives token usage, latency, success/failure metrics.                  |
| **Dead Letter Queue**          | Retry/Timeout edges + custom error callback to store failed tasks in DB        | Could persist failed state for later reprocessing.                    |

### 2. Mapping to **Temporal** (Workflow Orchestration Engine)

| **Your Component**               | **Temporal Concept**                                        | **Implementation Notes**                                                  |
| -------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------- |
| **API Gateway/Receptionist**     | HTTP API (FastAPI) that triggers a `WorkflowExecution`      | Use Temporal client SDK to start workflows and return `workflow_id`.      |
| **Orchestrator**                 | **Temporal Workflow** (stateful, resumable)                 | Defines main control flow, task DAG, and retry policies.                  |
| **Planner, Validator, Reviewer** | **Activities** within the workflow                          | Activities are idempotent & retryable.                                    |
| **Task Decomposition**           | Separate Activity executed first in workflow                | Output subtask DAG persisted in workflow state.                           |
| **Executor Service**             | Activities that call external APIs/tools                    | Temporal guarantees at-least-once execution but with idempotency keys.    |
| **Tool Broker/Registry**         | Activities + side-effect calls with approval gating         | Could block until human approval is received (human-in-the-loop pattern). |
| **Vector DB, State Storage**     | External databases accessed from activities                 | Temporal keeps workflow state but large artifacts go in PostgreSQL/Redis. |
| **Monitoring & Observability**   | Temporal Web UI + Prometheus metrics + OpenTelemetry traces | Native integration for observability and retries.                         |
| **Dead Letter Queue**            | Temporal's "Continue-As-New" + Failure Handling policies    | Failed workflows can be routed to a special "analysis workflow".          |

### 3. Mapping to **CrewAI** (Multi-Agent Orchestration Framework)

| **Your Component**             | **CrewAI Role/Concept**                                   | **Implementation Notes**                                             |
| ------------------------------ | --------------------------------------------------------- | -------------------------------------------------------------------- |
| **Task Decomposition Service** | **Lead Agent** ("Analyst")                                | Breaks user request into subtasks and assigns them to other agents.  |
| **Planner Service**            | **Strategist Agent**                                      | Generates structured plans based on retrieved context.               |
| **Plan Validator**             | **Reviewer Agent** (rule-focused, fast LLM)               | Approves or rejects plans before execution.                          |
| **Executor Service**           | **Worker Agents**                                         | Each executes a tool call or API request.                            |
| **Tool Broker & Registry**     | Tool registry is part of CrewAI's tool management         | Wrap your approval logic around tool calls.                          |
| **Reviewer Service**           | **QA Agent**                                              | Validates outputs before marking task as complete.                   |
| **LLM Abstraction Service**    | Model router (CrewAI supports multi-LLM setups)           | Route tasks to different models by complexity/cost.                  |
| **Orchestrator**               | CrewAI's Task Manager + Memory                            | Coordinates inter-agent communication & maintains shared scratchpad. |
| **Vector DB**                  | CrewAI Memory (long-term)                                 | Store successes/failures for retrieval by strategist agent.          |
| **Monitoring/Observability**   | Custom integration (CrewAI doesn't have full tracing yet) | Use callbacks to log events to Prometheus/Grafana.                   |

---

## What You Can Reuse vs. Build

### Can Largely Reuse:
- Orchestration & retries (Temporal)
- Multi-agent planner/executor logic (Orion, AutoGen, ModelScope-Agent)
- Tool registry & invocation (LangChain, llm-tools-hub)
- Vector database storage (Weaviate, Qdrant, Milvus)
- Observability & tracing (OpenTelemetry + Temporal UI)

### Must Build from Scratch / Extend Heavily:
- Plan validator (schema + rule enforcement)
- Tool broker policy enforcement & sandbox runtime
- Cost telemetry & token billing dashboard
- QA/reviewer service logic + learning loop
- Domain-specific business rules & governance layer

---

# Part 2: MVP Design

## MVP Architecture Overview

The MVP focuses on the core Plan → Execute → Review loop with minimal but production-ready components. This design validates the core workflow while providing a foundation for incremental enhancement.

```mermaid
graph TD
    U[User] --> API[FastAPI Gateway]
    API --> W[Temporal Workflow]
    W --> P[Planner Activity]
    W --> E[Executor Activity]
    W --> R[Reviewer Activity]
    
    P --> LLM[LLM Router]
    R --> LLM
    E --> TOOLS[Tool Registry]
    
    W --> PG[(PostgreSQL)]
    W --> REDIS[(Redis Cache)]
    
    subgraph "MVP Core"
        API
        W
        P
        E
        R
        LLM
        TOOLS
    end
    
    subgraph "Data Layer"
        PG
        REDIS
    end
```

## MVP Component Specifications

#### 1. FastAPI Gateway (Simplified)
**Purpose**: HTTP interface for task submission and status checking

**Endpoints**:
- `POST /tasks` - Submit new task
- `GET /tasks/{task_id}` - Get task status
- `GET /tasks/{task_id}/logs` - Get execution logs

**Features**:
- Basic authentication (API keys)
- Request validation with Pydantic
- Async task handling with 202 responses
- Simple rate limiting (per-user)

#### 2. Temporal Workflow (Core Orchestrator)
**Purpose**: Manages the Plan → Execute → Review loop with retries

**Workflow Logic (Conceptual)**:
```python
async def plan_execute_review_workflow(task_description: str):
    max_iterations = 3
    for attempt in range(max_iterations):
        # Plan
        plan = await workflow.execute_activity(
            generate_plan, task_description, 
            schedule_to_close_timeout=timedelta(minutes=2) # Example timeout
        )
        
        # Execute
        results = await workflow.execute_activity(
            execute_plan, plan,
            schedule_to_close_timeout=timedelta(minutes=10) # Example timeout
        )
        
        # Review
        review = await workflow.execute_activity(
            review_results, task_description, results,
            schedule_to_close_timeout=timedelta(minutes=1) # Example timeout
        )
        
        if review.status == "APPROVED":
            return {"status": "success", "results": results}
        elif review.status == "FAILED":
            return {"status": "failed", "error": review.feedback}
        
        # REVISE: Add feedback to task description for next iteration
        task_description += f"\nRevision needed: {review.feedback}"
    
    return {"status": "max_iterations_reached"}
```

#### 3. LLM Router (Multi-Provider)
**Purpose**: Route requests to appropriate models with fallback

**Model Strategy**:
- **Planner**: Gemini 2.5 Pro (primary) → GPT-4o (fallback)
- **Reviewer**: Gemini 2.5 Flash (primary) → GPT-4o-mini (fallback)
- **AI Tools**: Mistral Small (primary) → GPT-4o-mini (fallback)

**Implementation (Conceptual)**:
```python
class LLMRouter:
    async def call(self, model_type: str, prompt: str) -> str:
        providers = {
            "planner": ["gemini-2.5-pro", "gpt-4o"],
            "reviewer": ["gemini-2.5-flash", "gpt-4o-mini"],
            "ai_tool": ["mistral-small", "gpt-4o-mini"]
        }
        
        for provider in providers[model_type]:
            try:
                # Call specific provider logic
                return await self._call_provider(provider, prompt)
            except Exception as e:
                logger.warning(f"{provider} failed: {e}")
                continue
        raise Exception("All providers failed")
```

#### 4. Tool Registry (Essential Tools)
**Purpose**: Secure execution of predefined tools

**MVP Tools**:
- **Python Executor**: Sandboxed Python code execution
- **Web Search**: Search engine queries
- **File Operations**: Read/write local files
- **HTTP Client**: API calls to external services
- **Math Calculator**: Mathematical computations
- **Code Analyzer**: For static code analysis (e.g., call graph generation)

**Security**: Each tool runs in isolated Docker containers with resource limits

#### 5. Data Layer (PostgreSQL + Redis)
**Purpose**: State persistence and caching

**PostgreSQL Schema (Conceptual)**:
```sql
CREATE TABLE tasks (
    task_id UUID PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    result JSONB,
    error TEXT
);

CREATE TABLE executions (
    execution_id UUID PRIMARY KEY,
    task_id UUID REFERENCES tasks(task_id),
    plan JSONB NOT NULL,
    results JSONB,
    iteration INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Redis Usage**:
- LLM response caching (TTL: 1 hour)
- Rate limiting counters
- Active workflow state

### MVP Implementation Stack (Conceptual)

#### Backend Services (Python application structure)
```python
# FastAPI application structure
app/
├── main.py              # FastAPI app entry point
├── models/
│   ├── task.py         # Pydantic models for tasks
│   └── plan.py         # Pydantic models for plan schema validation
├── activities/
│   ├── planner.py      # Plan generation activity
│   ├── executor.py     # Plan execution activity
│   └── reviewer.py     # Result review activity
├── workflows/
│   └── main.py         # Temporal workflow definition
├── tools/
│   ├── python_exec.py  # Python execution tool implementation
│   ├── web_search.py   # Web search tool implementation
│   ├── file_ops.py     # File operations tool implementation
│   ├── code_analyzer.py# Code analysis tool implementation
│   └── base.py         # Base tool interface
├── llm/
│   └── router.py       # LLM routing logic
└── config.py           # Configuration management
```

#### Infrastructure (Docker Compose - Conceptual)
```yaml
version: '3.8'
services:
  api:
    build: . # Your FastAPI app
    ports: ["8000:8000"]
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/mvp
      - REDIS_URL=redis://redis:6379
      # LLM API keys should be injected securely (e.g., K8s secrets)
    depends_on: [postgres, redis, temporal]
    
  temporal:
    image: temporalio/auto-setup:latest # Temporal server and UI
    ports: ["7233:7233", "8080:8080"]
    
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=mvp
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes: ["postgres_data:/var/lib/postgresql/data"]
    
  redis:
    image: redis:7-alpine
    volumes: ["redis_data:/data"]

volumes:
  postgres_data:
  redis_data:
```

### MVP User Experience

#### Task Submission Flow (Conceptual)
```bash
# Submit task via API Gateway
curl -X POST http://localhost:8000/tasks \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Analyze the Python files in ./src and create a call graph from main() to process_data()",
    "constraints": {
      "max_duration": 300,
      "allowed_tools": ["python_exec", "file_operations", "code_analyzer"]
    }
  }'

# Expected Response
{
  "task_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "submitted",
  "estimated_completion": "2025-01-15T10:30:00Z" # Example
}
```

#### Status Monitoring (Conceptual)
```bash
# Check task status via API Gateway
curl http://localhost:8000/tasks/123e4567-e89b-12d3-a456-426614174000

# Expected Response (in progress)
{
  "task_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "executing",
  "current_step": "Analyzing Python files for function definitions",
  "progress": 0.6,
  "logs": [
    "Plan generated with 4 steps",
    "Executing step 1: List Python files in ./src",
    "Executing step 2: Parse AST for function definitions"
  ],
  "final_result": null # Will be populated upon completion
}
```

### MVP Success Criteria

#### Functional Requirements
- [ ] Accept task descriptions via REST API
- [ ] Generate structured execution plans
- [ ] Execute plans using available tools
- [ ] Review and iterate on results
- [ ] Return final results or failure reasons
- [ ] Handle at least 3 concurrent tasks
- [ ] Support task status monitoring
- [ ] Successfully execute the "code call sequence" task for simple Python projects.

#### Quality Requirements
- [ ] 95% success rate for well-defined tasks
- [ ] <60 seconds for simple tasks (file operations, basic analysis)
- [ ] <5 minutes for complex tasks (code analysis, multi-step workflows)
- [ ] Graceful failure handling with clear error messages
- [ ] Basic observability (logs, metrics)

### MVP Limitations and Future Enhancements

#### Known Limitations
- No comprehensive Plan Validator (relies on LLM to generate valid plans)
- Limited tool security (basic sandboxing only, actual code execution is an example; real-world needs stricter isolation)
- No learning from past executions (Vector DB not integrated for proactive learning in MVP)
- Single-tenant architecture
- Basic error handling and retry logic
- Limited language support for code analysis tool

#### Phase 2 Enhancements (Future)
- Plan Validator service with schema enforcement
- Vector Memory for learning from past executions
- Tool Broker with approval workflows
- Advanced monitoring and alerting
- Multi-tenant support with proper isolation
- Enhanced code analysis for multiple languages and larger codebases

### AI Model Selection and Cost Strategy (MVP Specific)

The balanced multi-provider approach optimizes for cost, reliability, and performance while avoiding vendor lock-in for the MVP.

#### Recommended Multi-Provider Setup for MVP
| **Component**                                  | **Recommended Provider**                                                                | **Reason / Tradeoff**                                                                                      |
| ---------------------------------------------- | --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **Planner (heavy reasoning, JSON generation)** | **Gemini 2.5 Pro** (Google)                                                             | Excellent structured output, large free tier (15–60 req/min, 1M tokens free in Vertex AI). Stable.         |
| **AI Tool (inside executor)**                  | **Mistral (small/medium)** via [Mistral API](https://mistral.ai) OR **Groq Llama 3 8B** | Mistral has generous free tier & low price after; Groq offers *ultra-low latency* (ideal for dev loops).   |
| **Reviewer (fast, cheap judgment)**            | **Gemini 2.5 Flash** or **Claude 3 Haiku**                                              | Flash is super cheap + fast, good for yes/no/approve-type tasks. Haiku is another fallback for robustness. |
| **Fallback / Cross-check (optional)**          | **OpenAI GPT-4o-mini**                                                                  | Use only when planner output fails validation. Good to have second opinion to reduce LLM hallucination.    |
| **Local Dev / Offline Mode**                   | **Ollama** (run Llama 3 / Phi-3 locally)                                                | No API cost, ideal for developers testing plans & executor pipelines offline.                              |

#### Cost & Quota Estimate (2025) for MVP
| Provider           | Free Tier                                  | Paid Price (approx)                    | Notes                               |
| ------------------ | ------------------------------------------ | -------------------------------------- | ----------------------------------- |
| Gemini 2.5 Pro     | 60 RPM free (1M tokens/month in Vertex AI) | ~$0.0015 / 1k input tokens           | Very generous. Perfect for Planner. |
| Gemini 2.5 Flash   | 60 RPM free (same quota as Pro)            | ~$0.0005 / 1k input tokens           | Cheap & fast.                       |
| Mistral API        | 5M tokens/month free (small model)         | ~$0.20 / 1M tokens                   | Great for heavy iterative loops.    |
| GroqCloud          | Free tier 10M tokens/month                 | Paid: ~$0.12 / 1M tokens (Llama3-8B) | Extremely fast (20-30ms responses). |
| OpenAI GPT-4o-mini | 50 free requests/day for many dev accounts | $0.15 / 1M tokens                     | Best as fallback only.              |

#### Practical Implementation Advice (Design Considerations)
1.  **LLM Abstraction Layer**: Keep your LLM abstraction layer (as in your architecture) so you can hot-swap providers without touching workflow logic.
2.  **Model Router Implementation**: Implement a model router that chooses provider based on:
    *   Step type (planner → Gemini, AI tool → Mistral/Groq)
    *   Current usage / quota left (fallback to OpenAI or local model if exceeded)
3.  **Simple Telemetry**: Add simple telemetry (tokens, latency, cost) → store in Temporal workflow history for future optimization.
4.  **Intelligent Caching**: Cache plans when running iterative dev tasks — avoid re-generating if same input.
5.  **Dynamic Fallback Logic**: Implement the conceptual fallback logic described in MVP Component Specifications (LLM Router).
6.  **Cost Optimization Strategies**:
    *   Batch processing: Group similar operations to reduce API overhead.
    *   Context compression: Use smaller context windows when possible.
    *   Model warming: Keep connections alive to reduce latency.
    *   Smart retries: Don't retry expensive calls immediately; use cheaper models first.

---

# Part 3: MVP Implementation

## MVP Workflow Implementation for Plan → Execute → Review → Loop

This implementation provides a lean but production-ready MVP solution using Temporal for orchestration and a multi-provider LLM router for flexibility and cost efficiency. It includes specific code for the Planner, Executor, and Reviewer activities, along with a custom "Code Analyzer" tool for the requested use case (call sequence tracing).

### Core Implementation (Python)

```python
# requirements:
# pip install temporalio google-generativeai mistralai openai

import asyncio, json, traceback
import os, ast
from collections import defaultdict, deque
from datetime import timedelta # Needed for Temporal timeouts

from temporalio import workflow, activity
import google.generativeai as genai
from mistralai.client import MistralClient
from openai import OpenAI

# === CONFIG ===
# Replace with actual API keys or environment variables
GEMINI_API_KEY = os.getenv("GEMINI_API_KEY", "your-gemini-key")
MISTRAL_API_KEY = os.getenv("MISTRAL_API_KEY", "your-mistral-key")
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY", "your-openai-key")

# Initialize clients (ensure API keys are valid)
try:
    if GEMINI_API_KEY:
        genai.configure(api_key=GEMINI_API_KEY)
    if MISTRAL_API_KEY:
        mistral_client = MistralClient(api_key=MISTRAL_API_KEY)
    if OPENAI_API_KEY:
        openai_client = OpenAI(api_key=OPENAI_API_KEY)
except Exception as e:
    print(f"Warning: Failed to initialize LLM client: {e}. Some models may not be available.")


# === MODEL ROUTER ===
async def call_ai_model(model_type: str, prompt: str, fallback=True):
    """
    Unified LLM call with fallback support based on model_type.
    model_type: "planner" | "reviewer" | "ai_tool"
    """
    workflow.logger.info(f"Calling LLM for {model_type} with prompt (first 100 chars): {prompt[:100]}")
    
    # Define providers for each model_type, ordered by preference (cost/performance)
    providers_config = {
        "planner": [("gemini-2.5-pro", "gemini"), ("gpt-4o", "openai")],
        "reviewer": [("gemini-2.5-flash", "gemini"), ("gpt-4o-mini", "openai")],
        "ai_tool": [("mistral-small", "mistral"), ("gpt-4o-mini", "openai")]
    }

    providers = providers_config.get(model_type, [])

    for model_name, provider_key in providers:
        try:
            if provider_key == "gemini":
                if not GEMINI_API_KEY: raise ValueError("Gemini API key not configured.")
                model = genai.GenerativeModel(model_name)
                response = model.generate_content(prompt)
                return response.text
            elif provider_key == "mistral":
                if not MISTRAL_API_KEY: raise ValueError("Mistral API key not configured.")
                resp = mistral_client.chat.complete(model=model_name, messages=[{"role":"user","content":prompt}])
                return resp.choices[0].message.content
            elif provider_key == "openai":
                if not OPENAI_API_KEY: raise ValueError("OpenAI API key not configured.")
                resp = openai_client.chat.completions.create(
                    model=model_name,
                    messages=[{"role":"user","content":prompt}]
                )
                return resp.choices[0].message.content
            else:
                raise ValueError(f"Unknown provider_key: {provider_key}")
        except Exception as e:
            workflow.logger.warning(f"LLM call to {model_name} failed: {e}. Trying fallback...")
            if not fallback:
                raise # If fallback is disabled, re-raise immediately
    
    # If all providers fail and fallback is true
    if fallback:
        workflow.logger.error(f"All primary LLM providers for {model_type} failed. No more fallbacks.")
    raise Exception(f"Failed to get LLM response for {model_type} after all attempts.")


# === TOOL: Python Code Analyzer ===
def analyze_call_graph(folder: str, start_func: str, end_func: str):
    """
    Analyzes Python files in a folder to build a simple call graph
    and find a path between a start and end function.
    """
    call_graph = defaultdict(list)
    func_locations = {} # To store file and line of function definitions

    # SECURITY: In a real system, folder access would be sandboxed.
    # This is an example for demonstration, assuming safe local access.
    
    if not os.path.exists(folder):
        return {"status": "error", "message": f"Folder '{folder}' not found."}

    for root, _, files in os.walk(folder):
        for f in files:
            if f.endswith(".py"):
                filepath = os.path.join(root, f)
                with open(filepath, "r", encoding="utf-8") as fp:
                    try:
                        source_code = fp.read()
                        tree = ast.parse(source_code, filename=filepath)
                        
                        for node in ast.walk(tree):
                            if isinstance(node, ast.FunctionDef):
                                func_name = node.name
                                func_locations[func_name] = f"{filepath}:{node.lineno}"
                                for sub_node in ast.walk(node):
                                    if isinstance(sub_node, ast.Call) and isinstance(sub_node.func, ast.Name):
                                        called_func_name = sub_node.func.id
                                        call_graph[func_name].append(called_func_name)
                    except Exception as e:
                        # Log parsing errors but continue
                        workflow.logger.warning(f"Skipping {filepath} due to parsing error: {e}")

    # BFS from start_func to end_func
    visited = set()
    queue = deque([(start_func, [start_func])]) # (current_func, current_path)

    while queue:
        current_func, path = queue.popleft()
        if current_func == end_func:
            return {"status": "success", "path": path, "func_locations": func_locations}
        
        # Avoid infinite loops for recursive calls or circular dependencies in graph
        if current_func in visited:
            continue
        visited.add(current_func)

        for neighbor in call_graph.get(current_func, []):
            if neighbor not in visited: # Only add if not already visited in this path
                queue.append((neighbor, path + [neighbor]))
    
    return {"status": "not_found", "path": [], "message": f"No call path found from {start_func} to {end_func}."}


# === TEMPORAL ACTIVITIES ===

# Activity for Plan Generation
@activity.defn
async def plan_activity(task: str):
    """Generates a structured plan in JSON format based on the task description."""
    prompt = f"""
    You are a highly skilled AI planner. Your goal is to create a step-by-step plan to accomplish the user's task.
    The task is: "{task}".

    If the task involves tracing a call sequence between two functions in a codebase,
    your plan must use the "code_analyzer" tool.
    The "code_analyzer" tool takes inputs: `folder` (path to codebase), `start_func`, `end_func`.

    Other general tools available:
    - "python_exec": Executes Python code. Inputs: `code` (string).
    - "web_search": Performs a web search. Inputs: `query` (string).
    - "file_operations": Reads/writes files. Inputs: `operation` ("read" or "write"), `path`, `content` (for write).
    - "http_client": Makes HTTP requests. Inputs: `method`, `url`, `headers`, `body`.
    - "math_calculator": Performs calculations. Inputs: `expression` (string).
    - "ai_tool": Use another LLM to answer a specific question. Inputs: `prompt` (string).

    Your plan must be in a JSON array format, where each object represents a step.
    Each step must have:
    - `step_id`: Unique identifier for the step (string).
    - `action`: Name of the tool to use (string, e.g., "code_analyzer", "python_exec", "ai_tool").
    - `inputs`: An object containing key-value pairs for the tool's parameters.
    - `outputs_expectation`: (Optional) A brief description of what you expect the output of this step to be.
    - `idempotency_key`: (Optional) A unique key to ensure the step is executed only once if retried.
    - `dependencies`: (Optional) An array of `step_id`s that must complete before this step can run.

    Example for code analysis:
    ```json
    {{
      "plan_id": "gen-plan-123",
      "task_id": "user-task-abc",
      "generated_at": "2025-01-01T12:00:00Z",
      "confidence": 0.95,
      "metadata": {{
        "priority": 1,
        "required_tools": ["code_analyzer"],
        "estimated_cost_tokens": 500,
        "estimated_duration_seconds": 60
      }},
      "steps": [
        {{
          "step_id": "analyze_codebase",
          "action": "code_analyzer",
          "inputs": {{
            "folder": "./src",
            "start_func": "main",
            "end_func": "process_data"
          }},
          "outputs_expectation": {{"path": ["main", "helper", "process_data"]}},
          "idempotency_key": "analyze_codebase_1"
        }},
        {{
          "step_id": "report_result",
          "action": "ai_tool",
          "inputs": {{
            "prompt": "Based on the path from 'analyze_codebase' tool, summarize the call sequence."
          }},
          "outputs_expectation": {{"summary": "Call sequence: main -> helper -> process_data"}},
          "dependencies": ["analyze_codebase"],
          "idempotency_key": "report_result_1"
        }}
      ]
    }}
    ```
    Your JSON response for the plan must conform to the Plan Contract Schema provided previously in the overall architecture.
    """
    try:
        plan_json_str = await call_ai_model("planner", prompt)
        # Attempt to parse and re-serialize to validate JSON structure
        plan_data = json.loads(plan_json_str)
        return json.dumps(plan_data, indent=2) # Return formatted JSON
    except Exception as e:
        workflow.logger.error(f"Plan generation failed: {e}")
        raise

# Activity for Plan Execution
@activity.defn
async def execute_activity(plan_json: str):
    """Executes the steps defined in the plan, invoking appropriate tools."""
    workflow.logger.info(f"Executing plan: {plan_json[:200]}...")
    try:
        plan = json.loads(plan_json)
    except json.JSONDecodeError:
        raise ValueError("Invalid plan JSON provided to executor.")
    
    results_per_step = {}
    
    # Simple dependency handling: just execute in order for MVP
    # For full DAG, a more complex scheduler would be needed
    for step in plan.get("steps", []):
        step_id = step.get("step_id", "unknown_step")
        action = step.get("action")
        inputs = step.get("inputs", {})
        
        workflow.logger.info(f"Executing step '{step_id}' with action '{action}'")

        try:
            if action == "python_exec":
                result = run_python_tool(inputs.get("code", ""))
            elif action == "ai_tool":
                result = await call_ai_model("ai_tool", inputs.get("prompt", ""))
            elif action == "code_analyzer":
                # Assuming local folder access for MVP, replace with secure access in production
                folder = inputs.get("folder")
                start_func = inputs.get("start_func")
                end_func = inputs.get("end_func")
                # SECURITY NOTE: In a real system, 'folder' should be validated/sandboxed.
                # For this MVP example, assume a relative path or pre-approved folder structure.
                result = analyze_call_graph(folder, start_func, end_func)
            elif action == "web_search":
                 # Placeholder for web search tool
                 result = {"status": "success", "result": f"Web search for '{inputs.get('query')}' (simulated)."}
            elif action == "file_operations":
                 # Placeholder for file operations
                 result = {"status": "success", "result": f"File operation '{inputs.get('operation')}' on '{inputs.get('path')}' (simulated)."}
            elif action == "http_client":
                 # Placeholder for HTTP client
                 result = {"status": "success", "result": f"HTTP {inputs.get('method')} to '{inputs.get('url')}' (simulated)."}
            elif action == "math_calculator":
                 # Placeholder for math calculator
                 try:
                    expr_result = eval(str(inputs.get('expression'))) # UNSAFE for production, for MVP only
                    result = {"status": "success", "result": expr_result}
                 except Exception as e:
                    result = {"status": "error", "error": str(e)}
            else:
                result = {"status": "error", "message": f"Unknown action: {action}"}
            
            results_per_step[step_id] = {"status": "completed", "output": result}
        except Exception as e:
            error_details = {"status": "failed", "error": str(e), "trace": traceback.format_exc()}
            workflow.logger.error(f"Step '{step_id}' failed: {e}", exc_info=True)
            results_per_step[step_id] = error_details
            # In a real system, depending on criticality, you might halt here or proceed
            # For MVP, we'll let review activity decide if it's a critical failure
            
    return json.dumps(results_per_step, indent=2)

# Activity for Reviewing Execution Results
@activity.defn
async def review_activity(task: str, execution_results: str):
    """Reviews the execution results against the original task objective."""
    workflow.logger.info(f"Reviewing results for task: {task[:100]}..., results: {execution_results[:200]}...")
    
    # Parse execution results to provide better context to the LLM
    try:
        results_data = json.loads(execution_results)
    except json.JSONDecodeError:
        results_data = {"parse_error": "Execution results are not valid JSON", "raw_results": execution_results}

    prompt = f"""
    You are an AI Quality Assurance specialist.
    The original task was: "{task}"

    Here are the execution results for the steps taken:
    {json.dumps(results_data, indent=2)}

    Based on the original task and the execution results, determine if the task was successfully completed.
    Specifically for the code call sequence task:
    - Check if the "code_analyzer" tool was used and if it reported "status": "success".
    - Verify if a "path" was found.

    Respond with a JSON object containing:
    - `status`: "APPROVED", "REVISE", or "FAILED".
    - `feedback`: A string explaining the decision, providing constructive criticism if "REVISE" or "FAILED".
    - `final_output`: (Optional) The extracted final output if status is "APPROVED".

    Example APPROVED for call sequence:
    ```json
    {{"status": "APPROVED", "feedback": "Successfully traced the call sequence.", "final_output": ["main", "step1", "step2", "process_data"]}}
    ```
    Example REVISE:
    ```json
    {{"status": "REVISE", "feedback": "The call graph analysis failed. Ensure the folder path is correct and accessible. Planner needs to regenerate the plan with a valid folder or error handling for tool."}}
    ```
    Example FAILED:
    ```json
    {{"status": "FAILED", "feedback": "Max iterations reached, and the task could not be completed successfully. The code analyzer consistently failed."}}
    ```
    """
    
    try:
        review_json_str = await call_ai_model("reviewer", prompt)
        review_data = json.loads(review_json_str)
        # Basic validation of review structure
        if "status" not in review_data or "feedback" not in review_data:
            raise ValueError("Review LLM did not return expected JSON structure.")
        return json.dumps(review_data, indent=2)
    except Exception as e:
        workflow.logger.error(f"Review generation or parsing failed: {e}", exc_info=True)
        # Fallback for critical review failure: assume revision needed
        return json.dumps({"status": "REVISE", "feedback": f"Review process failed: {e}. Planner needs to re-evaluate."})


# === TEMPORAL WORKFLOW ===
@workflow.defn
class PlanExecuteReviewWorkflow:
    @workflow.run
    async def run(self, task: str):
        workflow.logger.info(f"Starting workflow for task: {task}")
        max_loops = 3 # Limits iterations to prevent runaway costs/loops
        workflow_state = {"initial_task": task, "current_task_context": task, "history": []}

        for i in range(max_loops):
            workflow.logger.info(f"Workflow iteration {i+1}/{max_loops}")
            current_task_context = workflow_state["current_task_context"]

            try:
                plan_json = await workflow.execute_activity(
                    plan_activity, current_task_context, schedule_to_close_timeout=timedelta(seconds=30)
                )
                workflow.logger.info(f"Plan generated (first 100 chars): {plan_json[:100]}...")
            except Exception as e:
                workflow.logger.error(f"Plan activity failed in iteration {i+1}: {e}", exc_info=True)
                workflow_state["history"].append({"iteration": i+1, "stage": "plan", "status": "failed", "error": str(e)})
                # Attempt to revise the task context for planning in the next loop if possible
                workflow_state["current_task_context"] += f"\nPrevious planning attempt failed: {e}. Please try a different approach."
                continue # Skip execution and review, go to next iteration

            try:
                execution_results = await workflow.execute_activity(
                    execute_activity, plan_json, schedule_to_close_timeout=timedelta(seconds=60)
                )
                workflow.logger.info(f"Execution results (first 100 chars): {execution_results[:100]}...")
            except Exception as e:
                workflow.logger.error(f"Execute activity failed in iteration {i+1}: {e}", exc_info=True)
                workflow_state["history"].append({"iteration": i+1, "stage": "execute", "status": "failed", "error": str(e), "plan": plan_json})
                workflow_state["current_task_context"] += f"\nPrevious execution failed: {e}. Review the plan for correctness or tool issues."
                continue # Skip review, go to next iteration

            review_json = await workflow.execute_activity(
                review_activity, workflow_state["initial_task"], execution_results, schedule_to_close_timeout=timedelta(seconds=30)
            )
            review_data = json.loads(review_json)
            
            workflow_state["history"].append({
                "iteration": i+1,
                "stage": "review",
                "plan": plan_json,
                "execution_results": execution_results,
                "review_status": review_data.get("status"),
                "review_feedback": review_data.get("feedback")
            })

            if review_data.get("status") == "APPROVED":
                workflow.logger.info(f"Task APPROVED in iteration {i+1}. Final output: {review_data.get('final_output')}")
                return {"status": "completed", "final_output": review_data.get("final_output"), "history": workflow_state["history"]}
            elif review_data.get("status") == "REVISE":
                workflow.logger.info(f"Task requires REVISION in iteration {i+1}. Feedback: {review_data.get('feedback')}")
                workflow_state["current_task_context"] += f"\nRevision feedback: {review_data.get('feedback')}"
            else: # FAILED status from reviewer
                workflow.logger.warning(f"Task FAILED by reviewer in iteration {i+1}. Reason: {review_data.get('feedback')}")
                return {"status": "failed", "reason": review_data.get('feedback'), "history": workflow_state["history"]}
        
        workflow.logger.warning(f"Max iterations ({max_loops}) reached for task: {task}. Task remains incomplete.")
        return {"status": "failed", "reason": "Max iterations reached", "history": workflow_state["history"]}

```

### How This MVP Implementation Works

1.  **Configuration**: API keys for Gemini, Mistral, and OpenAI are loaded from environment variables.
2.  **LLM Router (`call_ai_model`)**:
    *   This function acts as your `LLM Abstraction Service`.
    *   It tries preferred models first (e.g., Gemini for `planner` and `reviewer`, Mistral for `ai_tool`).
    *   If a call fails (e.g., rate limit, API error, or missing API key), it automatically attempts to use a fallback model (e.g., OpenAI's GPT-4o-mini).
    *   It logs which model is being used and any failures.
3.  **Code Analyzer Tool (`analyze_call_graph`)**:
    *   This is a custom Python function acting as a "Tool".
    *   It uses Python's `ast` module to statically parse `.py` files in a given folder.
    *   It constructs a simple function call graph (function `A` calls function `B`).
    *   It performs a Breadth-First Search (BFS) to find a call path from a `start_func` to an `end_func`.
    *   **Security Note**: Directly executing arbitrary file system operations or `exec()` in a production environment is highly dangerous. For a real system, `analyze_call_graph` would interface with a securely sandboxed environment (e.g., a Docker container or a dedicated microservice with restricted permissions) to access the code.
4.  **Temporal Activities**:
    *   `plan_activity`: Calls the LLM Router with `model_type="planner"` and a detailed prompt to generate a JSON plan. It expects the plan to include steps using the `code_analyzer` tool if the task involves code analysis.
    *   `execute_activity`: Parses the `plan_json`. It iterates through steps, calling the appropriate local Python functions (like `run_python_tool`, `analyze_call_graph`) or `call_ai_model` for `ai_tool` steps.
    *   `review_activity`: Calls the LLM Router with `model_type="reviewer"` to evaluate the execution results against the original task. It's prompted to look specifically for the success/failure of the `code_analyzer` and the presence of a path.
5.  **Temporal Workflow (`PlanExecuteReviewWorkflow`)**:
    *   This is the core orchestrator.
    *   It runs a `max_loops` (e.g., 3) iteration loop.
    *   In each loop, it calls `plan_activity`, `execute_activity`, and `review_activity` sequentially.
    *   If the `review_activity` returns "APPROVED", the workflow completes.
    *   If "REVISE", the `task` description is updated with feedback, and the loop continues for a new plan generation.
    *   If "FAILED" or `max_loops` are reached, the workflow reports failure.
    *   `workflow.logger` is used for internal logging, visible in the Temporal Web UI.
    *   `timedelta` is used for `schedule_to_close_timeout` on activities, ensuring activities don't hang indefinitely.

### High-Level MVP Workflow

```mermaid
sequenceDiagram
    participant WF as Temporal Workflow
    participant LLM_P as LLM Router (Planner)
    participant LLM_R as LLM Router (Reviewer)
    participant LLM_A as LLM Router (AI Tool)
    participant EXE as Executor Activity
    participant CA as Code Analyzer Tool
    participant PY as Python Sandbox Tool
    participant U as User

    WF->>LLM_P: Prompt for Plan (task: "trace call sequence from A to B")
    LLM_P-->>WF: JSON Plan (with "code_analyzer" step)

    loop Up to 3 Iterations
        WF->>EXE: Execute Plan (e.g., {"action": "code_analyzer", "inputs": {...}})
        EXE->>CA: Call analyze_call_graph(folder, A, B)
        CA-->>EXE: Analysis Results (e.g., {"status": "success", "path": ["A", "X", "B"]})

        alt Plan also has AI tool steps
            EXE->>LLM_A: Call AI Tool (e.g., "Summarize path: A->X->B")
            LLM_A-->>EXE: AI Tool Response
        end

        EXE-->>WF: All Step Execution Results
        WF->>LLM_R: Prompt for Review (task, execution_results)
        LLM_R-->>WF: Review Result (e.g., {"status": "APPROVED", "feedback": "Path found"})

        alt Review APPROVED
            WF->>U: Task Completed Successfully
        else Review REVISE
            WF->>WF: Update Task Context with Feedback
        else Review FAILED (or Max Loops Reached)
            WF->>U: Task Failed
        end
    end
```
### MVP Use Case: Grep Code and Create Call Sequence

**Goal:** Given a folder with source code and user-provided start/end functions, output a call sequence (chain of function calls) connecting them.

**Example Task**: "Analyze the Python files in the `./src` folder and create a call sequence where the start function is `main` and the end function is `process_data`."

**How the MVP handles it:**

1.  **User Submission**: The user submits the task via the API Gateway (FastAPI).
2.  **Orchestrator (Temporal Workflow) Starts**: A `PlanExecuteReviewWorkflow` instance is initiated.
3.  **Planner Activity**:
    *   The `plan_activity` is executed.
    *   The `LLM Router` (using **Gemini 2.5 Pro**) receives a prompt guiding it to use the `code_analyzer` tool.
    *   The planner generates a JSON plan that includes a step with `action: "code_analyzer"` and `inputs: {"folder": "./src", "start_func": "main", "end_func": "process_data"}`.
4.  **Executor Activity**:
    *   The `execute_activity` is invoked with the generated plan.
    *   It identifies the `code_analyzer` step.
    *   It calls the `analyze_call_graph("./src", "main", "process_data")` function (which acts as your `Code Analyzer Tool`).
    *   The `analyze_call_graph` tool scans the Python files, builds a call graph, and attempts to find a path. Let's assume it finds `["main", "helper_function", "process_data"]`.
    *   The execution results are collected.
5.  **Reviewer Activity**:
    *   The `review_activity` is invoked with the original task and the execution results.
    *   The `LLM Router` (using **Gemini 2.5 Flash**) receives a prompt asking it to evaluate if the task (finding the call sequence) was successful based on the `code_analyzer`'s output.
    *   If `analyze_call_graph` returned `{"status": "success", "path": [...]}` and the path is valid, the reviewer responds with `{"status": "APPROVED", "feedback": "Successfully traced call sequence", "final_output": ["main", "helper_function", "process_data"]}`.
6.  **Workflow Completion**: The Temporal workflow receives the "APPROVED" status, marks the task as complete, and returns the final call sequence to the API Gateway for the user. If no path was found or an error occurred, the reviewer would request `REVISE` (to prompt the planner to try again, possibly with different parameters or by breaking down the problem further) or `FAILED`.

This MVP provides a functional, self-correcting loop for complex tasks like static code analysis using AI agents and robust orchestration.