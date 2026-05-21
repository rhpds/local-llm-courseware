# Business Case: Local LLM Courseware for RHDP Operations

Date: 2026-05-20
Status: Draft
Author: Joshua Disraeli

---

## Section 1: Defining the Opportunity and Problem Statement

### 1. Executive Summary

**Project Name:** Local LLM Courseware -- Hands-On Training for Local AI-Assisted Development

**Description and Objectives**

A four-module interactive training program that teaches the RHDP operations team how to run AI coding assistants against local models on their own hardware. The courseware guides learners through Ollama model management, Cursor IDE integration with local inference, RamaLama as a container-native alternative runtime, and LiteLLM proxy concepts for infrastructure understanding. All inference stays on-device -- no code leaves the laptop, no API keys required, no cloud dependency.

Core objectives:

- Standardize local AI tooling across the RHDP ops team (replacing ad hoc individual experimentation)
- Deliver a working Cursor + Ollama setup on every team member's machine, calibrated to their hardware
- Build infrastructure literacy around API translation proxies and container-native model serving
- Establish the "right tool for each scenario" framework: local models for private/offline work, Claude Code via Vertex for complex multi-file tasks

**Strategic Alignment**

This initiative directly aligns with Red Hat's stated AI strategy in three ways:

1. **Internal AI adoption at scale.** Red Hat CEO Matt Hicks reported at Summit 2026 that the company has scaled from 10 to nearly 200 production agents, with 85% of calls running on open-weight models hosted on Red Hat infrastructure. This courseware extends that momentum to the ops team by teaching self-hosted model workflows.

2. **Open-source AI tooling.** Red Hat's AI platform strategy centers on open models (Granite, Llama), open runtimes (vLLM, llama.cpp), and container-native delivery (RamaLama, Podman). Every tool in this courseware is open source and on the team's approved tools list.

3. **Developer productivity through AI.** Red Hat Developer Hub, Developer Lightspeed, and the broader multi-tool approach (VS Code + plugins, Cursor, Claude Code, Gemini CLI) all reflect a strategy of meeting developers where they work. This courseware operationalizes that strategy for the RHDP ops team specifically.

---

### 2. Problem Statement / Use Case

**Business Need**

The RHDP ops team writes and maintains Ansible automation, Python utilities, shell scripts, and infrastructure tooling daily. AI coding assistants can meaningfully accelerate this work -- but the team currently has no standardized approach. Some members have experimented individually with various tools; most have not. The team needs a structured onboarding path that accounts for the range of hardware (16GB to 64GB+ Macs) and provides a working local setup that respects data privacy constraints.

**Challenge / Inefficiency**

The original courseware attempted to solve this by routing Claude Code through a LiteLLM proxy to Ollama. This approach failed for fundamental reasons:

- **Performance:** A 30B model on an 18GB M3 Pro takes 30-40 seconds per response and exhausts system memory. Even a 14B model produces 29-37 second response times.
- **Quality:** Claude Code's system prompt includes complex tool definitions and a thinking parameter designed for Claude models. Local models cannot interpret these correctly, producing malformed JSON tool calls instead of usable answers.
- **Fragility:** LiteLLM's Anthropic passthrough endpoint does not respect `drop_params` or `model_info` settings for the thinking parameter. Workarounds are unreliable and break with upstream updates.

These are architectural mismatches, not configuration problems. The team needs the right tool for each scenario, not a workaround that produces a poor experience.

Meanwhile, the ad hoc experimentation pattern creates its own problems: inconsistent setups, no shared knowledge base, repeated troubleshooting of the same issues, and no hardware-aware model selection (leading to OOM crashes on lower-RAM machines).

**Impact of Inaction**

- 12 team members each spend 5-10 hours per week on tasks that AI coding assistants can accelerate. Without standardized local tooling, that productivity gain remains unrealized -- roughly 60-120 person-hours per week of potential improvement left on the table.
- Individual experimentation continues without guidance, leading to wasted time on setups that don't work (as proven by the original Module 01 experience with Claude Code + LiteLLM).
- The team falls behind Red Hat's broader AI adoption curve. At Summit 2026, every Red Hat organization -- including non-engineering teams -- was reported to have contributed code to the internal agent system. The ops team risks being the exception.

**Strategic Capability Mapping**

| Field | Detail |
|:------|:-------|
| **Business Capability** | RHDP Operations -- infrastructure automation, platform provisioning, tooling development |
| **Business Activity / Process** | Code generation, debugging, refactoring, documentation, and script authoring for Ansible playbooks, Python utilities, and shell automation |
| **Current Performance Metrics (Baseline)** | 5-10 hours/week per person spent on manually-authored code tasks. No standardized AI assistance in use. Ad hoc experimentation by individual team members with inconsistent results and no shared learnings. |

---

### Next Steps and Needs

- **Leadership visibility:** Review and approval from John Apple (manager) and Kadeem Lewis (project manager) to allocate team time for the four-module training program
- **Hardware audit:** Confirm RAM and chip specifications across all 12 team members' machines to validate the hardware-aware model selection table
- **Time allocation:** Approximately 1.5 hours per person for the full four-module courseware (20-25 minutes per module), plus 30 minutes for challenges
- **Tool access confirmation:** Verify that Cursor, Ollama, Podman Desktop, and RamaLama are all on the approved tools list and installable without IT escalation
- **Scheduling:** Coordinate a cohort-based rollout (recommended) or self-paced async delivery with a completion deadline

**References**

- [Redesign specification](docs/superpowers/specs/2026-05-20-local-llm-courseware-redesign.md) -- full technical design for the four-module structure
- [Red Hat Summit 2026: Scaling enterprise AI](https://www.redhat.com/en/blog/lab-ledger-scaling-enterprise-ai-red-hat-summit-2026) -- company-wide AI adoption metrics and direction
- [Forging the open path: Red Hat engineering AI adoption](https://www.redhat.com/en/blog/forging-open-path) -- internal multi-tool AI strategy
- [RamaLama: container-native AI models](https://www.redhat.com/en/blog/run-containerized-ai-models-locally-ramalama) -- Red Hat's container-native model runtime
- [Cursor + Ollama integration guide](https://dev.to/0xkoji/use-local-llm-with-cursor-2h4i) -- reference setup procedure
- [RamaLama GitHub repository](https://github.com/containers/ramalama)
- [Ollama model library](https://ollama.com/library)
- [Red Hat Developer Lightspeed](https://www.redhat.com/en/about/press-releases/red-hat-launches-red-hat-developer-lightspeed-ai-powered-developer-productivity) -- Red Hat's developer productivity AI portfolio
- [State of open source AI models in 2025](https://developers.redhat.com/articles/2026/01/07/state-open-source-ai-models-2025) -- landscape of locally-runnable models

---

## Section 2a: Strategic Decision -- Build vs. Buy Crosscheck

| Criteria | Answer |
|:---------|:-------|
| Is the proposed solution **competitive** to Red Hat product strategy? | **No.** This courseware teaches Red Hat-aligned tools (RamaLama, Granite models, Podman, Ollama) and reinforces the company's open-source AI strategy. It complements -- not competes with -- Red Hat AI, Developer Lightspeed, and OpenShift AI. |
| Is **custom LLM / data integration** a requirement? | **No.** The courseware uses off-the-shelf open models (Qwen, Granite) pulled from public registries. No fine-tuning, no custom training data, no proprietary datasets. |
| Is there any **additional cost** to Red Hat to deploy or utilize this tool/feature? | **No.** All tools are free and open source (Ollama, Cursor free tier, RamaLama, LiteLLM). Hardware is existing team-issued laptops. Courseware development is an internal effort. |
| Does an **out-of-the-box strategic vendor solution** answer the priority requirements? | **No.** No vendor sells "teach the RHDP ops team to use local AI tools on their specific hardware." This is internal enablement, not a product procurement decision. |

**Result:** All answers are No. No in-depth build vs. buy analysis is required. Proceeding to Section 3 with the recommended solution.

---

## Section 3: Proposed Solution and Business Case

### 5. Proposed Solution Overview

**AI Solution**

A four-module hands-on courseware delivered as interactive Claude Code skill commands that guide learners through setup, verification, and challenges:

| Module | Content | Duration |
|--------|---------|----------|
| 01: Ollama Setup + Model Management | Install Ollama, hardware-aware model selection (auto-detects RAM, recommends appropriate model size), model lifecycle commands | ~20 min |
| 02: Cursor + Ollama Integration | Connect Cursor IDE to local Ollama, configure endpoints, hands-on coding exercises, boundary documentation (what works, what doesn't) | ~25 min |
| 03: RamaLama Alternative Runtime | Container-native model serving via Podman, Granite model serving, runtime-agnostic frontend demonstration | ~20 min |
| 04: LiteLLM Proxy Concepts | API translation proxy setup, curl exercises across three API formats, multi-model routing, understanding why Claude Code + local models fails | ~20 min |

Hardware-aware model selection ensures every team member gets a working setup regardless of machine specs:

| RAM | Recommended Model | Size | Rationale |
|-----|-------------------|------|-----------|
| 16GB | qwen2.5-coder:7b | ~4.5 GB | Leaves ~11GB for OS/apps. Responsive. |
| 18GB | qwen2.5-coder:7b | ~4.5 GB | 14B proven too tight at 18GB in testing. |
| 24GB | qwen2.5-coder:14b | ~9 GB | Good balance of quality and headroom. |
| 32GB+ | qwen2.5-coder:14b | ~9 GB | Could go larger but diminishing returns. |
| 64GB+ | qwen3-coder:30b | ~18 GB | Enough headroom for the MoE model. |

**User Interaction**

A team member runs `/learn-01-ollama-setup` in Claude Code. The skill audits their current state (what's already installed, what's missing), walks them through each step with verification at each checkpoint, and ends with a hands-on challenge. Each module is self-contained -- if Ollama is already installed, the preflight detects it and skips ahead. The learner progresses through all four modules over 1-2 sessions, ending with a fully working local AI coding setup tailored to their hardware.

**Reusability and Synergy Assessment**

| Assessment Area | Detail |
|:----------------|:-------|
| **Internal Synergy** | Leverages existing approved tools (Ollama, Cursor, Podman Desktop, RamaLama). Module 03 directly teaches RamaLama, Red Hat's own container-native AI runtime. Module 04 builds LiteLLM proxy literacy that transfers to understanding API gateway patterns in OpenShift AI. The courseware platform itself (Claude Code skills) is the same pattern used across other team training initiatives. |
| **Reusability Potential** | The courseware structure and hardware-aware model selection logic are directly reusable for any Red Hat team onboarding to local AI tools. Module 03 (RamaLama) is particularly reusable -- any team with Podman Desktop can follow it. The "right tool for each scenario" framework (local for private work, cloud for complex tasks) applies organization-wide. Potential future audiences: other RHDP teams, consulting services, field engineering. |

---

### 6. Business Value and Impact

| Value Area | Detail |
|:-----------|:-------|
| **Time savings / hours saved** | Conservative estimate: 2-3 hours/week per person once proficient with local AI coding assistance (from a baseline of 5-10 hours/week on manually-authored tasks). At 12 team members, that is 24-36 hours/week reclaimed, or roughly 1,250-1,870 hours/year. |
| **Number of employees who benefit** | 12 (RHDP ops team). Potential expansion to adjacent RHDP teams. |
| **Estimated users** | 12 initial, with courseware reusable for broader audiences. |
| **Hard cost savings** | Training development is internal (no vendor cost). All tools are free/open source. No new hardware required -- uses existing team laptops. The only investment is development time for courseware authoring and team time for completion (~2 hours per person). |
| **Business criticality** | Supports Red Hat's enterprise-wide AI adoption initiative. Executive sponsors: John Apple (manager), Kadeem Lewis (project manager). Aligns with Summit 2026 mandate that every organization contributes to AI capability building. |
| **Intangibles** | Eliminates frustration from failed ad hoc setups. Builds shared vocabulary and baseline skills across the team. Establishes a repeatable onboarding pattern for new team members. Reduces dependency on cloud AI tools for privacy-sensitive or offline work. Teaches infrastructure concepts (API proxies, container-native serving) that transfer to production operations work. |

---

### 7. Audience Profile and Scale

| Field | Detail |
|:------|:-------|
| **Intended Audience / Persona** | RHDP Operations Engineers -- responsible for infrastructure automation (Ansible), Python tooling, shell scripting, and platform provisioning. Mix of senior and mid-level engineers, all on macOS, ranging from 16GB to 64GB+ RAM. |
| **Estimated User Volume (Initial)** | 12 -- the full RHDP ops team. |
| **Estimated User Volume (Scaled)** | 30-50 across broader RHDP organization if courseware proves effective. Courseware is open source and self-service, so organic adoption beyond the team is expected. |
| **Adoption Strategy** | Cohort-based rollout recommended: schedule a 2-hour block where the team works through all four modules together, enabling peer support and shared troubleshooting. Async self-paced option available for scheduling conflicts. Target: 100% team completion within 2 weeks of launch. |
| **User Interaction Frequency** | One-time training (1.5-2 hours). Ongoing daily use of the tools taught (Cursor + Ollama becomes a daily driver for local coding tasks). |

---

### 8. Resources Required

| Resource Type | Description | Quantity | Cost (Year 1) |
|:--------------|:------------|:---------|:---------------|
| **Personnel** | Courseware author (internal, existing headcount) | 1 | $0 incremental |
| **Personnel** | Learner time for 12 team members at ~2 hours each | 24 person-hours | $0 incremental (normal work hours) |
| **Technology** | Ollama, Cursor (free tier), RamaLama, LiteLLM, Podman Desktop | All free/open source | $0 |
| **Hardware** | Existing team-issued MacBooks (16GB-64GB+ RAM) | 12 | $0 (already provisioned) |
| **Models** | Qwen 2.5 Coder (7B/14B), Granite.code via RamaLama | Open-weight, pulled from public registries | $0 |
| **Model hosting** | Local only -- models run on each team member's laptop | N/A | $0 |

**Total incremental cost: $0.** This initiative requires time investment only, using existing headcount, hardware, and free open-source tooling.

---

### 9. Appendices

- **Redesign specification:** [`docs/superpowers/specs/2026-05-20-local-llm-courseware-redesign.md`](docs/superpowers/specs/2026-05-20-local-llm-courseware-redesign.md) -- full technical design including problem analysis, module structure, hardware-aware model selection, and decision log
- **Repository:** [rhpds/local-llm-courseware](https://github.com/rhpds/local-llm-courseware) -- courseware source, module content, and setup scripts
- **Existing Module 01 (pre-redesign):** Demonstrates the LiteLLM + Claude Code approach and documents why it was abandoned
- **Testing results:** 18GB M3 Pro testing showed 29-37 second response times with 14B model, OOM conditions with 30B model, and malformed JSON tool calls from local models attempting to parse Claude Code's system prompt

**Key references for strategic context:**

- [Red Hat Summit 2026: Scaling enterprise AI](https://www.redhat.com/en/blog/lab-ledger-scaling-enterprise-ai-red-hat-summit-2026)
- [Forging the open path: Red Hat engineering AI adoption](https://www.redhat.com/en/blog/forging-open-path)
- [Red Hat AI platform overview](https://www.redhat.com/en/whats-new-red-hat-ai)
- [RamaLama: container-native AI](https://www.redhat.com/en/blog/run-containerized-ai-models-locally-ramalama)
- [Red Hat Developer Lightspeed](https://www.redhat.com/en/about/press-releases/red-hat-launches-red-hat-developer-lightspeed-ai-powered-developer-productivity)
- [Granite models overview](https://www.redhat.com/en/topics/ai/what-are-granite-models)
- [State of open source AI models 2025](https://developers.redhat.com/articles/2026/01/07/state-open-source-ai-models-2025)
