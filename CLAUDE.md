# MOT History MCP Server

## Project Status

This project is in the planning phase. The implementation plan exists in `mot_mcp_plan.md` (gitignored) but no source files have been created yet. The project needs to be built from scratch following the plan.

## Project Overview

Python MCP server wrapping the UK MOT History API using FastMCP. Provides MOT data (results, mileage, defects, recalls) conversationally via MCP tools.

## Tech Stack

- **Python 3.10+** with **uv** for package management
- **FastMCP** via `mcp[cli]` (official SDK, `from mcp.server.fastmcp import FastMCP`)
- **httpx** for async HTTP
- **Pydantic** for data models
- **python-dotenv** for env config
- **hatchling** as build backend

## Target Project Structure (not yet created)

```
mot_api_agents/
├── pyproject.toml          # deps: mcp[cli], httpx, python-dotenv, pydantic; build: hatchling
├── .env.example            # template for MOT API credentials
├── .gitignore
├── README.md
└── src/mot_mcp/
    ├── __init__.py
    ├── server.py           # FastMCP server + 4 tool handlers + main()
    ├── client.py           # TokenManager, MotApiClient, exceptions
    ├── models.py           # Pydantic models (API responses + computed outputs)
    └── analysis.py         # compute_reliability_summary() — pure logic, no I/O
```

### Implementation Order

1. `pyproject.toml` + `.env.example` + `src/mot_mcp/__init__.py` (scaffold)
2. `models.py` — Pydantic models for API responses and computed outputs
3. `client.py` — OAuth 2.0 token manager + async HTTP client
4. `analysis.py` — reliability computation (pure logic, no I/O)
5. `server.py` — FastMCP tools wiring it all together
6. `README.md` — usage docs

## Key Patterns

### FastMCP Server Setup

```python
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("MOT History")

@mcp.tool()
async def my_tool(param: str) -> str:
    """Docstring becomes tool description. Args section describes params."""
    return result

def main():
    mcp.run(transport="stdio")
```

### Lifespan Pattern for Shared HTTP Client

Use a lifespan async context manager to share the `MotApiClient` across all tool invocations:

```python
from contextlib import asynccontextmanager
from mcp.server.fastmcp import FastMCP, Context

@asynccontextmanager
async def app_lifespan(server: FastMCP):
    async with create_client() as client:
        yield {"client": client}

mcp = FastMCP("MOT History", lifespan=app_lifespan)

@mcp.tool()
async def my_tool(reg: str, ctx: Context) -> str:
    client = ctx.request_context.lifespan_context["client"]
    ...
```

### OAuth 2.0 Token Management

- POST to `https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token`
- Cache token, auto-refresh 60s before expiry
- Env vars: `MOT_API_TENANT_ID`, `MOT_API_CLIENT_ID`, `MOT_API_CLIENT_SECRET`, `MOT_API_SCOPE`, `MOT_API_KEY`

### Error Handling in Tools

- 429 -> `RateLimitError` (24hr lockout if daily limit exceeded — be careful)
- 404 -> `VehicleNotFoundError`
- 401/403 -> retry token then `AuthenticationError`
- All tools return user-friendly error messages, not raw exceptions

## Commands (once project is scaffolded)

- `uv sync` — install dependencies
- `uv run mot-mcp` — run the server (entry point: `mot-mcp = "mot_mcp.server:main"`)
- `mcp dev src/mot_mcp/server.py` — run MCP inspector for development/testing

## API Reference

| Detail | Value |
|--------|-------|
| Base URL | `https://history.mot.api.gov.uk` |
| Auth | OAuth 2.0 client credentials + `X-API-Key` header |
| Rate limit | 15 req/sec, 500K/day, burst 10 |
| Registration endpoint | `/v1/trade/vehicles/registration/{reg}` |
| VIN endpoint | `/v1/trade/vehicles/vin/{vin}` |

## MCP Tools

1. **`lookup_vehicle_by_registration`** — Full MOT history by reg
2. **`lookup_vehicle_by_vin`** — Full MOT history by VIN
3. **`check_mot_status`** — Quick MOT status, expiry, recalls
4. **`get_vehicle_reliability_summary`** — Computed reliability analysis

## Guidelines

- Never commit `.env` files — credentials must stay local
- Tools return formatted text, not raw JSON
- Normalize registration input: uppercase, strip spaces
- All HTTP calls must be async via httpx
- Use Pydantic models for all API response parsing
- Rate limiting is strict — implement proper backoff and never exceed 15 req/sec
