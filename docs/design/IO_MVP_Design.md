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