---
name: model-gateway
description: >
  Use this skill when configuring LLM API calls through model gateways
  and routers. Triggers include "call an LLM", "set up OpenRouter",
  "configure model gateway", "add LLM to my agent", "switch models",
  "set up LLM factory", "create LLM client", "use LangChain with
  OpenRouter", or any task involving LLM API configuration, model
  selection, gateway routing, streaming, or provider-agnostic LLM access.
---

# Model Gateway Skill

A skill that governs how to **configure**, **call**, and **troubleshoot** LLM API calls through model gateways (OpenRouter, LiteLLM, or direct provider APIs).

Identify which operation applies, then follow the corresponding section.

---

## Operation: CONFIGURE — Setting Up an LLM Gateway

### Pre-Configuration Checklist

- [ ] I know the **gateway provider** (OpenRouter, LiteLLM, direct API)
- [ ] I have the **API key** and know where it's stored (`.env`, secrets manager)
- [ ] I know the **default model** to use
- [ ] I know if the project uses **LangChain**, raw HTTP, or a provider SDK

### Environment Variables Pattern

Store all LLM configuration in `.env` — never hardcode keys or URLs.

```env
# Gateway configuration
OPEN_ROUTER_GATEWAY_API_BASE_URL=https://openrouter.ai/api/v1
OPEN_ROUTER_GATEWAY_API_KEY=sk-or-v1-your-key-here

# Model selection (provider/model-name format for OpenRouter)
LAB_AGENT_MODEL=anthropic/claude-sonnet-4.6

# Optional: tracing
LANGSMITH_API_KEY=your-langsmith-key
LANGSMITH_TRACING_V2=true
LANGSMITH_PROJECT=your-project-name
```

**Always provide a `.env.example`** with placeholder values and comments — never commit real keys.

### LLM Factory Pattern (Recommended)

Create a single shared factory that all agents import. This centralizes configuration and makes model switching trivial.

```python
"""
Shared LLM configuration and factory.

Usage:
    from common.llm import create_llm
    llm = create_llm()                          # uses default model
    llm = create_llm(model_name="google/gemini-2.5-flash")  # override model
"""

import os
from pathlib import Path
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI

# Load .env from project root
_project_root = Path(__file__).resolve().parent.parent.parent
_env_file = _project_root / ".env"
if not _env_file.exists():
    _env_file = Path.cwd() / ".env"
load_dotenv(_env_file, override=True)

# Config from environment
GATEWAY_BASE_URL = os.getenv(
    "OPEN_ROUTER_GATEWAY_API_BASE_URL", "https://openrouter.ai/api/v1"
)
GATEWAY_API_KEY = os.getenv("OPEN_ROUTER_GATEWAY_API_KEY", "")
DEFAULT_MODEL = os.getenv("LAB_AGENT_MODEL", "anthropic/claude-sonnet-4.6")


def create_llm(model_name=None, temperature=0.0):
    """Create a ChatOpenAI instance routed through the model gateway."""
    model_name = model_name or DEFAULT_MODEL
    if not GATEWAY_API_KEY:
        raise ValueError(
            "OPEN_ROUTER_GATEWAY_API_KEY must be set in .env"
        )
    return ChatOpenAI(
        model=model_name,
        temperature=temperature,
        api_key=GATEWAY_API_KEY,
        base_url=GATEWAY_BASE_URL,
        max_retries=3,
    )
```

### Why This Pattern Works

- **Single source of truth** — one factory, one `.env`, all agents use the same config
- **Model switching** — change `LAB_AGENT_MODEL` in `.env` to switch all agents at once, or pass `model_name` per-agent for different models
- **Gateway-agnostic** — swap `BASE_URL` to switch from OpenRouter to LiteLLM or a direct provider without code changes
- **LangChain compatible** — `ChatOpenAI` with a custom `base_url` works with any OpenAI-compatible gateway

---

## Operation: CALL — Making LLM Calls

### Basic Call (LangChain)

```python
from common.llm import create_llm

llm = create_llm()
response = llm.invoke("Explain TCP handshake in one sentence")
print(response.content)
```

### With Structured Messages

```python
from langchain_core.messages import SystemMessage, HumanMessage

llm = create_llm()
response = llm.invoke([
    SystemMessage(content="You are a network engineer."),
    HumanMessage(content="Explain TCP handshake"),
])
```

### With LangChain Prompt Templates

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a {role}."),
    ("human", "{question}"),
])
chain = prompt | create_llm()
response = chain.invoke({"role": "network engineer", "question": "Explain TCP"})
```

### With Tool Calling

```python
from langchain.tools import tool

@tool
def search_logs(query: str, severity: str = "all") -> str:
    """Search application logs for entries matching the query."""
    # implementation
    ...

llm = create_llm()
llm_with_tools = llm.bind_tools([search_logs])
response = llm_with_tools.invoke("Find all error logs from today")
```

### Streaming

```python
llm = create_llm()
for chunk in llm.stream("Explain VoLTE registration"):
    print(chunk.content, end="", flush=True)
```

### Different Models for Different Agents

```python
# Fast model for classification
classifier = create_llm(model_name="google/gemini-2.5-flash", temperature=0.0)

# Strong model for complex reasoning
analyzer = create_llm(model_name="anthropic/claude-sonnet-4.6", temperature=0.0)

# Creative model for report writing
writer = create_llm(model_name="anthropic/claude-sonnet-4.6", temperature=0.7)
```

### Model Selection Guide

| Use Case | Recommended Approach |
|---|---|
| Classification, routing | Fast + cheap model, temperature 0 |
| Analysis, reasoning | Strong model, temperature 0 |
| Code generation | Strong model, temperature 0 |
| Creative writing | Any model, temperature 0.5-0.8 |
| Summarization | Mid-tier model, temperature 0 |

### OpenRouter Model Naming

Models follow the `provider/model-name` format:

```
anthropic/claude-sonnet-4.6
openai/gpt-5.2
google/gemini-2.5-flash
meta-llama/llama-4-maverick
mistralai/mistral-large
```

Check available models at the gateway's model list endpoint:
```bash
curl https://openrouter.ai/api/v1/models -H "Authorization: Bearer $KEY"
```

---

## Operation: TROUBLESHOOT — Diagnosing LLM Call Issues

### Diagnostic Checklist

Check in this order:

1. **API key set?** — `echo $OPEN_ROUTER_GATEWAY_API_KEY` (should not be empty)
2. **Base URL correct?** — must end with `/api/v1` for OpenRouter
3. **Model name valid?** — must use `provider/model-name` format
4. **Credits available?** — check gateway dashboard for balance
5. **Rate limited?** — check for 429 status codes in response
6. **Model supports feature?** — not all models support tool calling or JSON mode

### Common Errors

| Error | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Invalid or missing API key | Check `OPEN_ROUTER_GATEWAY_API_KEY` in `.env` |
| `402 Payment Required` | Insufficient credits | Add credits at gateway dashboard |
| `404 Not Found` | Invalid model name | Use `provider/model-name` format, check model list |
| `429 Rate Limited` | Too many requests | Add retry logic with exponential backoff |
| `502 Bad Gateway` | Upstream provider error | Retry or use fallback model |
| `timeout` | Model too slow or input too long | Reduce input size, use faster model, increase timeout |
| Empty response | Model returned nothing | Check `max_tokens`, check if input makes sense |

### Enabling Tracing (LangSmith)

Add to `.env` for full call visibility:

```env
LANGSMITH_API_KEY=lsv2_sk_your-key
LANGSMITH_TRACING_V2=true
LANGSMITH_PROJECT=your-project
```

This logs every LLM call — input, output, latency, tokens, cost — without code changes.

### Fallback Models

For production reliability, configure fallback models:

```python
def create_llm_with_fallback(primary=None, fallback=None, **kwargs):
    """Create LLM with automatic fallback on failure."""
    primary_llm = create_llm(model_name=primary, **kwargs)
    if fallback:
        fallback_llm = create_llm(model_name=fallback, **kwargs)
        return primary_llm.with_fallbacks([fallback_llm])
    return primary_llm
```

---

## Core Rules

1. **Never hardcode API keys** — always use environment variables via `.env`
2. **Always provide `.env.example`** — document every variable with placeholder values
3. **Use a shared factory** — one `create_llm()` function, not scattered API configs
4. **Set temperature to 0** for deterministic tasks (classification, extraction, analysis)
5. **Use the right model for the job** — fast/cheap for routing, strong for reasoning
6. **Add retry logic** — LLM APIs are unreliable; use `max_retries` at minimum
7. **Enable tracing** — LangSmith or equivalent for debugging and cost tracking
8. **Test with multiple models** — verify your prompts work across providers

---

## What NOT To Do

- Hardcode API keys in source code
- Create separate LLM clients in every file (use a shared factory)
- Use expensive models for simple classification tasks
- Skip retry logic (APIs fail regularly)
- Ignore token costs (monitor via tracing or gateway dashboard)
- Assume all models support all features (tool calling, JSON mode, vision)
- Commit `.env` files to version control
