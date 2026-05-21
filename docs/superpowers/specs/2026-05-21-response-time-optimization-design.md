# Response Time Optimization Design

**Date**: 2026-05-21
**Status**: Approved
**Scope**: Reduce TTFT and total generation time across all Showroom pages on CPU-only hardware

## Problem

The local LLM stack (Ollama -> LiteLLM -> Showroom) runs on CPU-only OCP nodes. Both time-to-first-token and total generation time are noticeably slow, making the courseware feel sluggish. The current config uses all defaults -- no Ollama parallelism tuning, no flash attention, no caching, and a single model for all tasks.

## Solution

Three complementary changes applied together:

### 1. Fast Model Addition (qwen3:0.6b)

Add qwen3:0.6b (523 MB, 40K context) alongside the existing qwen3:1.7b. Route simple pages (system prompts, structured output) to the fast model. Keep complex pages (RAG, tool use) on the full model.

**Model routing by page:**

| Page | Attribute | Model |
|------|-----------|-------|
| 01-04 (original) | `{litellm_model}` | qwen3:1.7b |
| 05 (System Prompts) | `{litellm_model_fast}` | qwen3:0.6b |
| 06 (Structured Output) | `{litellm_model_fast}` | qwen3:0.6b |
| 07 (RAG) | `{litellm_model}` | qwen3:1.7b |
| 08 (Tool Use) | `{litellm_model}` | qwen3:1.7b |
| 09 (MCP Server) | N/A (read-only walkthrough) | N/A |

Each page's intro paragraph mentions which model it uses (e.g., "This section uses `qwen3:0.6b`...").

**Infrastructure:**
- Pull both models sequentially in the model-pull job (sequential to avoid OOM)
- Add second entry to LiteLLM model_list
- Add `litellm_model_fast` to Showroom user_data and antora.yml attributes
- New default variable: `ocp_workload_local_llm_lab_ollama_model_fast: "qwen3:0.6b"`

**Memory budget:** qwen3:0.6b (~523MB) + qwen3:1.7b (~1.5GB) + KV cache overhead fits within existing 8Gi memory limit.

### 2. Ollama CPU Tuning

Add environment variables to the Ollama deployment:

| Variable | Value | Purpose |
|----------|-------|---------|
| `OLLAMA_FLASH_ATTENTION` | `1` | Faster attention computation, reduced memory during inference |
| `OLLAMA_NUM_PARALLEL` | `2` | Allow 2 concurrent inference requests (default: 1, serializes) |
| `OLLAMA_KEEP_ALIVE` | `10m` | Keep models loaded 10 minutes after last request (default: 5m) |

No resource limit changes needed. Existing 2-4 CPU cores and 4-8Gi memory are sufficient.

### 3. LiteLLM In-Memory Caching

Enable LiteLLM's built-in local cache in the configmap:

```yaml
litellm_settings:
  cache: true
  cache_params:
    type: "local"
    ttl: 600
```

- Cache key: SHA-256 of model + messages + temperature + params
- Storage: in-memory, max 200 items, max 1MB per item
- TTL: 600 seconds (10 minutes) -- long enough for a lab session, short enough to not go stale
- No Redis or external dependency required

**Why this works for courseware:** Every student runs the same curl commands with identical prompts. After the first execution, subsequent runs return cached responses instantly.

## Files Changed

| File | Change |
|------|--------|
| `deploy/agnosticd-role/.../defaults/main.yml` | Add fast model variable |
| `deploy/agnosticd-role/.../templates/ollama-deployment.j2` | Add 3 env vars (flash attention, parallel, keep-alive) |
| `deploy/agnosticd-role/.../templates/litellm-configmap.j2` | Add second model_list entry + litellm_settings cache block |
| `deploy/agnosticd-role/.../templates/model-pull-job.j2` | Pull both models sequentially |
| `deploy/showroom-content/content/antora.yml` | Add `litellm_model_fast` attribute |
| `deploy/showroom-content/.../pages/05-system-prompts-and-temperature.adoc` | Swap `{litellm_model}` to `{litellm_model_fast}`, add model name to intro |
| `deploy/showroom-content/.../pages/06-structured-output.adoc` | Same as above |
| `deploy/showroom-content/.../pages/07-rag-chat-with-your-docs.adoc` | Add model name to intro (no attribute change) |
| `deploy/showroom-content/.../pages/08-tool-use.adoc` | Add model name to intro (no attribute change) |

## What Stays the Same

- Pages 01-04: untouched (already use `{litellm_model}`)
- Page 09: untouched (read-only walkthrough)
- Resource limits: unchanged (both models fit in existing 8Gi)
- Route timeout: unchanged (300s)
- LiteLLM image version: unchanged (v1.85.1)

## Expected Impact

| Scenario | Before | After |
|----------|--------|-------|
| Simple Q&A (pages 5-6) | 5-15s TTFT, 20-40s total | ~2-5s TTFT, 8-15s total |
| Complex tasks (pages 7-8) | 5-15s TTFT, 30-60s total | ~4-12s TTFT, 25-50s total |
| Repeated identical request | Same as first request | ~0ms (cache hit) |
| Concurrent users (2 simultaneous) | Serialized, 2x wait | Parallel, ~1x wait |

Estimates based on CPU-only inference with flash attention enabled. Actual numbers depend on cluster node specs.

## Testing

1. Deploy changes to live cluster
2. Time a baseline request on page 05 before changes
3. Apply Ollama env vars, restart pod, time same request
4. Pull qwen3:0.6b, update LiteLLM config, time page 05 with fast model
5. Enable caching, run same request twice, verify second is instant
6. E2E Playwright run across all 9 pages to verify no regressions
