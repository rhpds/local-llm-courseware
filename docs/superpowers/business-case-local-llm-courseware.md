# Business Case: Local LLM Courseware for RHDP Operations

Date: 2026-05-20
Status: Draft
Author: Joshua Disraeli

---

## Executive Summary

**Local LLM Courseware** is a four-module, ~90-minute training program that gives the 12-person RHDP ops team a working local AI coding setup -- Cursor + Ollama on their own hardware, calibrated to their machine specs. No code leaves the laptop, no API keys, no cloud dependency. All tools are approved and open source.

**Strategic alignment:** Red Hat scaled to 200 production agents with 85% open-weight models (Summit 2026). This courseware extends that momentum to the ops team using Red Hat-aligned tools (RamaLama, Granite, Podman, Ollama).

## Problem

The team spends 5-10 hours/week per person on code tasks that AI assistants can accelerate, but has no standardized local tooling. Ad hoc experimentation produces inconsistent setups and wasted time. The original approach -- Claude Code through a LiteLLM proxy to Ollama -- failed due to 30-40s response times, malformed tool calls, and thinking parameter incompatibility. These are architectural mismatches, not configuration problems.

**Impact of inaction:** 60-120 person-hours/week of potential improvement unrealized. The team falls behind Red Hat's company-wide AI adoption curve.

## Solution

Four hands-on modules with hardware-aware model selection (auto-detects RAM, recommends 7B/14B/30B):

| Module | Content | Duration |
|--------|---------|----------|
| 01: Ollama Setup | Install, hardware-aware model selection, model management | ~20 min |
| 02: Cursor + Ollama | IDE integration, coding exercises, boundary documentation | ~25 min |
| 03: RamaLama | Red Hat's container-native runtime, Granite model serving | ~20 min |
| 04: LiteLLM Concepts | API translation proxies, multi-format exercises | ~20 min |

## Business Value

| Metric | Detail |
|:-------|:-------|
| **Time reclaimed** | 1,250-1,870 hours/year (2-3 hrs/week per person across 12 engineers) |
| **Total cost** | $0 incremental -- all tools free/open source, existing hardware |
| **Investment** | ~2 hours per person for training, internal courseware authoring |
| **Reusability** | Directly reusable for any Red Hat team; RamaLama module works for anyone with Podman |
| **Scale** | 12 initial, 30-50 across broader RHDP if effective |

## Build vs. Buy

Not competitive to Red Hat product strategy. No custom LLM/data integration. No additional cost. No vendor solution addresses this need. **No in-depth analysis required.**

## Next Steps

- Leadership review: John Apple (manager), Kadeem Lewis (project manager)
- Hardware audit across 12 team machines
- Cohort-based rollout (~2 hour block) or self-paced with 2-week completion target

## References

- [Redesign spec](docs/superpowers/specs/2026-05-20-local-llm-courseware-redesign.md) -- full technical design
- [Repo: rhpds/local-llm-courseware](https://github.com/rhpds/local-llm-courseware)
- [RHDPOPS-23490](https://redhat.atlassian.net/browse/RHDPOPS-23490) -- Jira task
- [Red Hat Summit 2026: Scaling enterprise AI](https://www.redhat.com/en/blog/lab-ledger-scaling-enterprise-ai-red-hat-summit-2026)
- [Forging the open path: Red Hat engineering AI adoption](https://www.redhat.com/en/blog/forging-open-path)
- [RamaLama: container-native AI](https://www.redhat.com/en/blog/run-containerized-ai-models-locally-ramalama)
- [Red Hat Developer Lightspeed](https://www.redhat.com/en/about/press-releases/red-hat-launches-red-hat-developer-lightspeed-ai-powered-developer-productivity)
