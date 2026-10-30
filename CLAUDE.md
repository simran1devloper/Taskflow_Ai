Act as a Principal Software Architect, Senior Backend Engineer, AI Engineer, and Engineering Manager.

We are starting a new project called TaskFlow AI from a blank repository.

I have already created CLAUDE.md, which contains project-wide engineering instructions.

IMPORTANT: We are NOT starting implementation yet.

Your responsibility is to conduct a comprehensive engineering discovery and architecture review before creating the master implementation plan.

### 1. Understand the product
Analyze the project idea and identify:
- Core problem being solved.
- Target users.
- Primary user journeys.
- Functional requirements.
- Non-functional requirements.
- MVP scope versus future enhancements.

### 2. Challenge my assumptions
Do not blindly accept my proposed technology stack or architecture.

Evaluate:
- React + TypeScript.
- Python + FastAPI.
- SQLite versus PostgreSQL.
- SQLAlchemy.
- LangChain versus LangGraph versus direct LLM integration.
- Ollama versus hosted LLM APIs.

Explain the trade-offs and recommend the simplest production-oriented choices.

### 3. System architecture
Propose:
- High-level architecture.
- Component responsibilities.
- Frontend-backend communication.
- Database schema and relationships.
- AI orchestration architecture.
- API boundaries.
- Error handling and logging strategy.

Include Mermaid architecture diagrams where useful.

### 4. Agentic AI architecture
Evaluate whether we actually need multiple agents.

Identify:
- Which capabilities should be deterministic workflows.
- Which capabilities genuinely require LLM reasoning.
- Agent tools and permissions.
- Human approval requirements.
- Memory and context management.
- Retry and failure handling.
- AI evaluation strategy.

### 5. Engineering quality
Identify:
- Security risks.
- Data consistency issues.
- Scalability bottlenecks.
- Testing strategy.
- Observability requirements.
- Potential architectural overengineering.

### 6. Harness Engineering strategy
Recommend how we should use:
- CLAUDE.md
- PLAN.md
- Feature-specific plans.
- Skills.
- Specialized subagents.
- Hooks.
- MCP servers.
- Automated tests.
- Git checkpoints.

Explain which capabilities are necessary for the MVP and which should be introduced later.

### 7. Conduct an interactive discovery session

Do not immediately generate PLAN.md.

First, present:
1. Your understanding of the project.
2. Missing requirements.
3. Architectural assumptions.
4. Major technical decisions requiring approval.
5. Potential risks and alternative approaches.

Then interview me with focused questions, one at a time.

Challenge weak decisions. Explain why a question matters before asking it.

Do not make assumptions about important product requirements.

### Final deliverable

After all critical questions have been resolved, prepare an Architecture Decision Record (ADR) containing the approved technical decisions.

Only after my explicit approval should you generate PLAN.md.

Do not modify application source code or create implementation files during discovery.