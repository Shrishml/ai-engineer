# AI Engineer Roadmap — 80/20 (20% that yields 80%)

## 1. Git & GitHub

| Command | Use |
|---------|-----|
| `clone`, `pull`, `push` | Daily sync |
| `branch`, `checkout`, `switch` | Feature isolation |
| `add`, `commit`, `amend` | Staging + fixing last commit |
| `merge`, `rebase` | Integrate branches |
| `stash`, `reset`, `revert` | Undo / context-switch |
| `cherry-pick` | Move specific commits |
| `log`, `diff`, `status` | Inspect state |

**GitHub**: PRs, code review, branch protection rules, `.gitignore`, `CODEOWNERS`.

---

## 2. API Development (FastAPI)

### Core
- `@app.get/post/put/delete` — path/query/body params, Pydantic models
- Dependency injection (`Depends`) — reuse auth, DB sessions, config
- Middleware (CORS, logging, rate-limit)
- **Auth**: OAuth2 password flow, JWT (`python-jose`), `OAuth2PasswordBearer`
- **Multi-worker**: `uvicorn --workers N` or `gunicorn + uvicorn.workers.UvicornWorker`
- Structured logging, error handlers, `HTTPException`

### SSE (Server-Sent Events) & WebSocket

| | SSE | WebSocket |
|---|-----|-----------|
| Direction | Server → client only | Bidirectional |
| Transport | HTTP (long-lived GET) | Upgrade from HTTP → ws:// |
| Use case | Streaming LLM tokens, status updates | Chat, real-time collab, live dashboards |
| FastAPI | `StreamingResponse` + `text/event-stream` | `from fastapi import WebSocket` |

```python
# SSE — streaming LLM response
@router.get("/stream")
async def stream():
    async def event_stream():
        async for chunk in llm.astream(prompt):
            yield f"data: {chunk}\n\n"
        yield "data: [DONE]\n\n"
    return StreamingResponse(event_stream(), media_type="text/event-stream")

# WebSocket
@router.websocket("/ws")
async def ws(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        await websocket.send_text(f"Echo: {data}")
```

### Principles
- RESTful naming: `/users/{id}` not `/getUser`
- Statelessness, idempotent writes (PUT is idempotent, POST is not)
- Pagination, filtering, sorting on list endpoints
- Proper HTTP status codes
- OpenAPI auto-docs (`/docs`)

---

## 3. Containers & Orchestration

### Docker
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
- Build: `docker build -t image:tag .`
- Push to **ACR**: `az acr login`, `docker push <registry>/image:tag`
- `.dockerignore`, multi-stage builds for smaller images

### Kubernetes
- **Pod**: smallest deployable unit (1+ containers)
- **Deployment**: replicas, rolling updates, resource limits
- **Service**: stable IP/DNS + load balancing across pods
- **ConfigMap / Secret**: inject config and credentials
- `kubectl apply/get/describe/logs/exec/port-forward/delete`

---

## 4. CI/CD

### GitHub Actions
```yaml
on: push                 # trigger
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt
      - run: pytest
  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - run: docker build -t image .
      - run: docker push <registry>/image
      - run: kubectl set image deployment/app app=image
```

### Harness
- Pipeline-as-code, approval gates, canary/blue-green deployments
- Integrates with K8s, ACR, GitHub

---

## 5. Agents & LLM Tooling

### Prompting
- System prompt + user prompt structure
- Few-shot, chain-of-thought, structured output (JSON mode)
- Token limits, context window management

### MCP (Model Context Protocol)
- **FastMCP**: `from fastmcp import FastMCP` — expose tools as MCP server
- Tools = Python functions decorated with `@mcp.tool()`
- Resources = data sources (files, DB queries)
- Connect Claude Code to any MCP server for agentic workflows

### Claude Agent SDK + Claude Code
- Use Claude Code as the agent harness
- Configure DeepSeek API as backend via settings.json / env vars (because deepseek is dirt cheap)
- Custom slash commands, hooks (pre/post tool events)
- Skills system for reusable agent behaviors

### LangChain / LangGraph
- **LangChain**: Chains, tools, memory, retrievers (RAG) - Just top level how it works 
- **LangGraph**: Stateful agent graphs — nodes + conditional edges
- Key pattern: `graph.add_node("agent", call_model)` → `add_conditional_edges("agent", should_continue)`

---

## 6. Scheduling & Async Patterns

| Concept | Meaning |
|---------|---------|
| **Scheduled agent runs** | Cron / Celery Beat / APScheduler trigger agents on a timer |
| **Polling** | Client repeatedly checks status (simple but chatty) |
| **Callback URL (webhook)** | Server POSTs result when ready (efficient, needs exposed endpoint) |
| **Idempotency** | Same request → same result. Use idempotency keys, dedup on receipt. Critical for payment/state-change agents |

---

## 7. Databases

### Postgres
- Run locally: `docker run -e POSTGRES_PASSWORD=... -p 5432:5432 postgres:16`
- `psql` or `SQLAlchemy` + `asyncpg` in FastAPI
- **CRUD**: `SELECT/INSERT/UPDATE/DELETE`

### Indexing
```sql
CREATE INDEX idx_user_email ON users(email);           -- equality lookup
CREATE INDEX idx_orders_created ON orders(created_at);  -- range queries
```
- B-tree (default), GIN for full-text, partial indexes for filtered queries
- `EXPLAIN ANALYZE` to verify index usage
- Trade-off: faster reads, slower writes

---

## 8. Evals

- Unit-test agent outputs: expected answer, tool calls made, latency < threshold
- Regression suites: known inputs → assert output quality
- LLM-as-judge: one model scores another's response
- Metrics: accuracy, faithfulness, tool-selection precision, hallucination rate

---

## 9. JS Basics (Enough to Read/Critique Frontend)

| Term | What it is |
|------|-----------|
| Component | Reusable UI block (React/Vue/Svelte) |
| Props | Data passed into a component |
| State | Mutable data inside a component |
| Hook | `useState`, `useEffect` — React's way to manage state/side-effects |
| JSX | HTML-in-JS syntax |
| Virtual DOM | React's diff engine |

---

## 10. Code Quality

- **Modular**: one file = one responsibility. Split routes, services, models, config
- **Clean**: descriptive names, no magic numbers, consistent formatting (ruff/black)
- **Maintainable**: type hints everywhere, docstrings on public APIs, no dead code
- **Extendable**: dependency injection, abstract base classes, config over hardcoding
- Lint: `ruff`, `mypy`. Format: `black`. Pre-commit hooks.
