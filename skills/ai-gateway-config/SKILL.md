---
name: ai-gateway-config
description: Unity Gateway (AI Gateway) configuration — model routing, traffic splitting, fallbacks, rate limits, guardrails, and inference table logging. Read this skill when the user asks to create or configure AI Gateway endpoints, set up model routing, or apply guardrails to LLM access.
---

# AI Gateway Configuration

## What is Unity Gateway?
Unity Gateway (formerly AI Gateway) provides a unified, governed interface for LLM access. It handles routing, rate limiting, guardrails, and audit logging for all AI model calls. In the CoE program, all AI access goes through the Gateway — never direct.

## Creating an AI Gateway Endpoint
```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Create an AI Gateway endpoint
w.ai_gateway.create_endpoint(
    name="coe-llm-gateway",
    description="CoE program — governed LLM access for all Q3 AI accelerators",
    rate_limits=[{
        "key": "per_user",
        "renewal_period": "minute",
        "calls": 60,
        "tokens": 100000,
    }],
    guardrails={
        "input": {
            "valid": True,
            "filter": ["pii", "toxic", "violence"],
        },
        "output": {
            "valid": True,
            "filter": ["pii", "toxic"],
        },
    },
)
```

## Model Routing with Traffic Splitting
```python
# Route traffic between primary and fallback models
w.ai_gateway.create_model_route(
    endpoint_name="coe-llm-gateway",
    name="primary-route",
    routes=[
        {"model": "databricks-llama-4-maverick", "weight": 80},
        {"model": "databricks-claude-3-7-sonnet", "weight": 20},
    ],
)
```

## Fallback Configuration
```python
w.ai_gateway.update_endpoint(
    name="coe-llm-gateway",
    fallback=[{
        "model": "databricks-claude-3-7-sonnet",
        "max_attempts": 2,
    }],
)
```

## Inference Table Logging
```python
# Enable inference table for audit logging
w.ai_gateway.update_endpoint(
    name="coe-llm-gateway",
    inference_table_spec={
        "enabled": True,
        "catalog_name": "coe_tpch",
        "schema_name": "ai_logs",
        "table_name_prefix": "inference",
    },
)
```

## Querying the Gateway (from notebooks/apps)
```python
import mlflow.deployments

client = mlflow.deployments.get_deploy_client("databricks")
response = client.predict(
    endpoint="coe-llm-gateway",
    inputs={
        "messages": [{"role": "user", "content": "Summarize the customer churn risk for customer 12345"}],
        "max_tokens": 500,
    },
)
```

## CoE Context
Q3 accelerator `ai_gateway_setup` creates a governed LLM access endpoint for all Q3 AI workloads. The Q1 DAB already defines Gateway variables: `gateway_endpoint`, `guardrail_model`, `guardrail_fallback_model`, `rate_limit_rpm`, `rate_limit_tpm`.

## Q1 Integration: Guardrail Enforcement
The Q1 governance KB defines guardrail policies. The AI Gateway enforces them:
- **PII filter**: Blocks prompts containing direct identifiers (names, SSNs, phone numbers)
- **Toxicity filter**: Blocks harmful content in both input and output
- **Rate limiting**: Per-user limits prevent runaway costs
- **Inference logging**: All LLM calls logged to `coe_tpch.ai_logs` for audit trails (Q4 compliance)

## Best Practices
- **Never bypass the Gateway**: All LLM access must go through the Gateway for governance
- **Enable inference tables**: Required for Q4 audit trail compliance
- **Use ordered fallbacks**: Primary model → fallback model → error
- **Scope rate limits per user, not per endpoint**: Prevents one user from consuming all capacity
- **Test guardrails with known PII**: Verify that guardrails catch actual PII patterns