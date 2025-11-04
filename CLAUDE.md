# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Evo AI is an open-source platform for creating and managing AI agents. It supports multiple agent frameworks (Google ADK, CrewAI) and protocols (A2A, MCP), enabling complex agent orchestration and interoperability.

## Development Commands

### Backend Commands

```bash
# Setup and installation
make venv                                      # Create virtual environment
source venv/bin/activate                       # Activate virtual environment
make install-dev                               # Install with development dependencies

# Running the application
make run                                       # Start development server (http://localhost:8000)
make run-prod                                  # Start production server with multiple workers

# Database operations
make alembic-upgrade                           # Apply pending migrations
make alembic-revision message="description"    # Create new migration
make alembic-downgrade                         # Revert last migration
make alembic-reset                             # Reset database to initial state
make seed-all                                  # Run all database seeders

# Code quality
make lint                                      # Run flake8 linting
make format                                    # Format code with black

# Docker operations
make docker-build                              # Build all Docker services
make docker-up                                 # Start all Docker services
make docker-seed                               # Seed database in Docker
```

### Frontend Commands

```bash
cd frontend                                    # Navigate to frontend directory
pnpm install                                   # Install dependencies
pnpm dev                                       # Start development server (http://localhost:3000)
pnpm build                                     # Build for production
pnpm start                                     # Start production server
pnpm lint                                      # Run ESLint
```

## Architecture Overview

### Multi-Agent Framework Architecture

Evo AI is built around a flexible agent composition system that supports 7 agent types:

1. **LLM Agent** - Direct language model interaction with tools
2. **Sequential Agent** - Executes sub-agents in order
3. **Parallel Agent** - Executes sub-agents simultaneously
4. **Loop Agent** - Repeats sub-agents up to N iterations
5. **A2A Agent** - Implements Google's Agent 2 Agent protocol for interoperability
6. **Workflow Agent** - Uses LangGraph for complex state-based orchestration
7. **Task Agent** - Structured task execution with defined goals

**Critical insight**: Agents are composed recursively - any agent can have sub-agents stored in `config["agent_tools"]` as a list of UUIDs. The `AgentBuilder` recursively builds sub-agents and composes them using the appropriate pattern (sequential, parallel, etc.).

### Agent Execution Flow

**Location**: `/src/services/adk/agent_builder.py` and `/src/services/adk/agent_runner.py`

```
User Request → API Route → AgentRunner.run_agent()
    → AgentBuilder.build_llm_agent()/build_sequential_agent()/etc.
        → Recursively build sub-agents from agent_tools UUIDs
        → Load MCP servers and custom tools
        → Compile all tools into agent
    → Execute agent with streaming via SSE
    → Handle file uploads/downloads
    → Return streamed response
```

### Service-Oriented Architecture

The backend follows a strict layered architecture:

- **API Layer** (`/src/api/`) - FastAPI routes with JWT authentication
- **Service Layer** (`/src/services/`) - Business logic, isolated from HTTP concerns
- **Model Layer** (`/src/models/models.py`) - SQLAlchemy ORM models
- **Schema Layer** (`/src/schemas/`) - Pydantic validation and serialization

**Key pattern**: All routes use dependency injection for database sessions and authentication. Services handle all business logic and return domain objects, not HTTP responses.

### Multi-Tenancy & Authorization

The platform implements client-based multi-tenancy:

- Every `User` belongs to a `Client`
- JWT tokens contain `client_id` for all requests
- Resources (agents, API keys, sessions) are scoped to clients
- Admin users can impersonate clients for support
- `jwt_middleware.py` enforces client isolation on all protected routes

### Agent Configuration Storage

Agent configuration is stored as JSON in the `Agent.config` field. Structure varies by agent type:

**LLM Agent config**:
```json
{
  "model": "gpt-4",
  "temperature": 0.7,
  "max_tokens": 2000,
  "agent_tools": ["uuid1", "uuid2"],  // Sub-agents
  "mcp_servers": [{"id": "server1", "name": "filesystem"}],
  "custom_tools": ["tool1"]
}
```

**Workflow Agent config** (additional fields):
```json
{
  "flow": {
    "nodes": [...],  // LangGraph node definitions
    "edges": [...]   // State transitions
  }
}
```

### MCP (Model Context Protocol) Integration

**Location**: `/src/services/adk/mcp_service.py`

MCP servers provide tools to agents. Two connection types:

1. **Stdio** - Local MCP servers (e.g., `npx @modelcontextprotocol/server-filesystem`)
2. **SSE** - Remote MCP servers via HTTP with Server-Sent Events

**Discovery**: `mcp_discovery.py` provides utilities for discovering available MCP servers. MCP servers are registered in the database and can be shared across agents.

**Tool Loading**: When building an agent, the `MCPService` loads all tools from configured MCP servers and compiles them into the agent's toolset.

### A2A (Agent 2 Agent) Protocol

**Location**: `/src/api/a2a_routes.py`, `/src/utils/a2a_enhanced_client.py`

Evo AI implements the full A2A specification:

- **Agent Card**: `/.well-known/agent.json` endpoint exposes agent capabilities
- **Messaging**: JSON-RPC 2.0 protocol with `message/send` and `message/stream` methods
- **Client**: `EnhancedA2AClient` auto-detects A2A endpoints and handles protocol negotiation
- **Bidirectional**: Evo AI agents can call other A2A agents, and external agents can call Evo AI agents

**Usage**: Create an A2A agent that references another agent's URL. The runner uses the A2A client to communicate.

### LangGraph Workflow Orchestration

**Location**: `/src/services/adk/custom_agents/workflow_agent.py`

Workflow agents use LangGraph's `StateGraph` for complex orchestration:

- Define nodes for each sub-agent
- Define conditional edges based on state
- Manage shared state across agent interactions
- Prevent cycles and infinite loops
- Persist workflow state in sessions

**Key difference from Sequential/Parallel**: Workflows support conditional branching and state-dependent routing between agents.

### API Key Encryption

**Location**: `/src/services/apikey_service.py`, `/src/utils/crypto.py`

API keys for LLM providers are encrypted at rest:

- Environment variable `ENCRYPTION_KEY` must be set
- Keys are encrypted using Fernet symmetric encryption
- Decryption only happens when executing agents
- Each agent can have its own API key, or inherit from client defaults

### Langfuse Tracing & Observability

**Location**: `/src/utils/otel.py`

OpenTelemetry integration provides comprehensive tracing:

- Set `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, and `OTEL_EXPORTER_OTLP_ENDPOINT`
- Automatically traces agent executions, LLM calls, and tool invocations
- View traces in Langfuse dashboard for debugging and monitoring

## Frontend Architecture

### Next.js 15 App Router Structure

The frontend uses Next.js 15's App Router (`/frontend/app/`):

- **File-based routing** - Each `page.tsx` creates a route
- **Layouts** - `layout.tsx` and `client-layout.tsx` provide nested layouts
- **React Server Components** - Default component type for improved performance
- **Client components** - Use `"use client"` directive when needed

### Key Frontend Modules

- **Agents** (`/app/agents/`) - Complex agent management UI with forms for each agent type
  - Forms in `/app/agents/forms/` handle agent creation/editing
  - Dialogs in `/app/agents/dialogs/` for API keys, tools, sharing
  - Workflow builder in `/app/agents/workflows/` uses ReactFlow for visual editing

- **Chat** (`/app/chat/`) - Real-time chat interface
  - WebSocket connection for streaming responses
  - File upload/download support
  - Session management and history

- **Services** (`/frontend/services/`) - API client layer
  - `api.ts` provides Axios instance with JWT interceptors
  - Service files wrap API endpoints with TypeScript types
  - Automatic token refresh and logout on 401

### State Management

- **React Context** - Global state for auth, theme, impersonation
- **React Hook Form** - Form state management with Zod validation
- **Local state** - Component state with useState/useReducer
- **No global state library** - Intentionally avoiding Redux/Zustand for simplicity

## Code Standards & Conventions

### Language Requirements

**ALL code must be written in English**:
- Comments, docstrings, log messages
- Variable names, function names, class names
- API error messages
- Commit messages

### Commit Message Format

Use Conventional Commits:
```
<type>(<scope>): <description>

Types: feat, fix, docs, style, refactor, perf, test, chore
Examples:
  feat(agents): add workflow agent type
  fix(api): correct validation error in agent creation
  refactor(services): improve agent builder modularity
```

### Python Code Style

- **Indentation**: 4 spaces
- **Line length**: Maximum 79 characters
- **Formatting**: Black (enforced via `make format`)
- **Linting**: Flake8 (enforced via `make lint`)
- **Type hints**: Use for all function signatures
- **Docstrings**: Required for all public functions and classes

### Backend Patterns

**Services**:
- Handle all business logic
- Use `SQLAlchemyError` for error handling
- Always rollback on error
- Log errors with context
- Return domain objects, not HTTP responses

**Routes**:
- Use appropriate HTTP status codes (200, 201, 204, 400, 401, 403, 404)
- Validate input with Pydantic schemas
- Use `HTTPException` for errors
- Implement pagination for list endpoints
- Use `async def` for all route handlers

**Migrations**:
- Use descriptive names: `alembic revision -m "add workflow agent support"`
- Test upgrades AND downgrades
- Use CASCADE for foreign keys when appropriate

### Frontend Patterns

**Components**:
- Prefer functional components with hooks
- Use TypeScript for all components
- Separate concerns: UI components vs. container components
- Keep components small and focused

**API Calls**:
- Always use services from `/frontend/services/`
- Handle errors consistently
- Show loading states
- Use React Query for caching when beneficial

## Critical Configuration

### Environment Variables

**Backend** (`.env`):
```bash
# Required
POSTGRES_CONNECTION_STRING="postgresql://..."
REDIS_HOST="localhost"
JWT_SECRET_KEY="your-secret-key"
ENCRYPTION_KEY="your-encryption-key"

# Optional but recommended
AI_ENGINE="adk"  # or "crewai"
EMAIL_PROVIDER="sendgrid"  # or "smtp"
LANGFUSE_PUBLIC_KEY="pk-lf-..."
LANGFUSE_SECRET_KEY="sk-lf-..."
OTEL_EXPORTER_OTLP_ENDPOINT="https://cloud.langfuse.com/api/public/otel"
```

**Frontend** (`frontend/.env`):
```bash
NEXT_PUBLIC_API_URL="http://localhost:8000"
```

### Switching Agent Frameworks

Set `AI_ENGINE` environment variable:
- `"adk"` - Google Agent Development Kit (default, production-ready)
- `"crewai"` - CrewAI framework (experimental)

The codebase has separate implementations in `/src/services/adk/` and `/src/services/crewai/`.

## Testing

Currently limited test coverage. When adding tests:

- Backend tests in `/src/tests/`
- Use pytest: `pytest src/tests/`
- Test structure mirrors source structure
- Use fixtures for database setup
- Mock external services (LLM APIs, MCP servers)

## Common Development Patterns

### Adding a New Agent Type

1. Add type to `Agent.type` enum validation in `models.py`
2. Create agent class in `/src/services/adk/custom_agents/`
3. Add builder method in `agent_builder.py`
4. Add execution logic in `agent_runner.py`
5. Create frontend form in `/frontend/app/agents/forms/`
6. Update TypeScript types in frontend
7. Create migration: `make alembic-revision message="add new agent type"`

### Adding a New Tool

Tools can be:
1. **MCP Server Tools** - Register MCP server in database, tools auto-discovered
2. **Custom Tools** - Defined in `/src/services/adk/custom_tools.py`
3. **Agent Tools** - Other agents used as tools (sub-agents)

### Debugging Agent Execution

1. Enable Langfuse tracing (set environment variables)
2. Check backend logs (structured logging via `logger.py`)
3. Use `/docs` endpoint to test API directly
4. Check session records in database for execution history
5. Inspect `Agent.config` JSON for configuration issues

## Database Schema Key Points

- **Multi-tenancy**: `client_id` foreign key on most tables
- **Agent composition**: `agent_tools` array in config JSON stores sub-agent UUIDs
- **Folders**: Agents organized in folders via `folder_id` foreign key
- **API Keys**: Encrypted at rest, linked to clients or agents
- **Sessions**: Track conversation history with agents
- **Audit logs**: Track administrative actions

## API Documentation

Interactive API docs available at:
- Swagger UI: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc

All endpoints under `/api/v1/` prefix require JWT authentication (except auth endpoints).

## Security Considerations

- JWT tokens expire (configurable via `JWT_EXPIRATION_MINUTES`)
- Passwords hashed with bcrypt
- API keys encrypted with Fernet
- Email verification required for new accounts
- Account lockout after failed login attempts
- Client isolation enforced at middleware level
- Admin impersonation logged in audit trail

## Troubleshooting

**Database connection issues**: Verify PostgreSQL is running and connection string is correct
**Redis connection issues**: Check Redis is running on configured port
**Agent execution failures**: Check Langfuse traces and backend logs
**MCP server issues**: Verify npx is available for stdio servers, check network for SSE servers
**Frontend API errors**: Verify `NEXT_PUBLIC_API_URL` matches backend URL
**Migration conflicts**: Use `make alembic-reset` to reset database (development only)
