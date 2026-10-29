---
name: mcp-server-config
description: Model Context Protocol (MCP) server creation, tool registration, AI Gateway fronting, and Unity Catalog registration for MCP services. Read this skill when the user asks to create or configure MCP servers, register tools, or expose MCP services through Unity Gateway.
---

# MCP Server Configuration

## What is MCP?
Model Context Protocol (MCP) lets you register external tool servers that Databricks AI agents (Genie, Agent Bricks) can call. MCP servers are registered in Unity Catalog and fronted by AI Gateway for governance.

## Creating an MCP Server in Unity Catalog
```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Register an external MCP server
w.ai_gateway.create_mcp_server(
    name="coe_tpch.tools.data_quality_checker",
    description="Check data quality on TPC-H tables",
    server_url="https://mcp-server.example.com/sse",
    auth_type="OAUTH",  # or API_KEY
    tools=[
        {
            "name": "check_nulls",
            "description": "Check for null values in a specified table column",
            "input_schema": {
                "type": "object",
                "properties": {
                    "table_name": {"type": "string"},
                    "column_name": {"type": "string"},
                },
                "required": ["table_name"],
            },
        },
    ],
)
```

## MCP Server Architecture
```
Agent (Genie/Agent Bricks)
    ↓
AI Gateway (rate limits, guardrails, routing)
    ↓
MCP Server (Unity Catalog registered)
    ↓
Tools (check_nulls, get_schema, run_query)
```

## Granting Access to MCP Servers
```sql
-- Grant access to a user or group
GRANT EXECUTE ON MCP SERVER coe_tpch.tools.data_quality_checker TO `data-engineering-team`;
```

## Querying Available MCP Servers
```python
servers = w.ai_gateway.list_mcp_servers()
for s in servers:
    print(f"{s.name}: {s.description}")
    for tool in s.tools:
        print(f"  - {tool.name}: {tool.description}")
```

## CoE Context
Q3 accelerator `mcp_server_setup` demonstrates creating an MCP server with data quality tools that operate on TPC-H tables. The MCP server is fronted by AI Gateway for governance and registered in `coe_tpch.tools` schema.

## Best Practices
- **Always front MCP servers with AI Gateway**: Governance is not optional — rate limits and guardrails apply
- **Use descriptive tool names and schemas**: Agents need clear descriptions to select the right tool
- **Scope tools by domain**: Separate servers for data quality vs. governance vs. analytics
- **OAuth over API key**: Use OAuth authentication when possible for better audit trails
- **Register in Unity Catalog**: MCP servers must be UC-registered for access control