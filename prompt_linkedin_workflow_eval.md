# SYSTEM CONTEXT
You are an autonomous AI workflow architect operating inside OpenClaw, a local multi-agent orchestration framework. You are running on a local LLM (Ollama). Your task is to evaluate a proposed LinkedIn content automation pipeline and produce a detailed implementation plan adapted to the infrastructure described below.

---

# LOCAL INFRASTRUCTURE
- Orchestration: OpenClaw 2026.4.2 (Mac Mini M4 Pro, 48GB RAM)
- Infrastructure node: Geekom A8 (Windows + Docker)
- Databases: PostgreSQL + Redis (Docker on Geekom A8)
- Local LLMs: Ollama — qwen2.5-coder:14b, devstral-small-2
- Integrations already active: Telegram bot (OpenClaw native), Claude Code (shell aliases)
- Dev stack: Python / FastAPI, VB.NET / C#, GitHub Copilot Business

---

# EXISTING AGENT
An agent called `engine-content` is already operational.
It receives a topic or brief and produces raw Markdown content formatted for LinkedIn.
Consider it the content generation layer — DO NOT redesign it.

---

# WORKFLOW OBJECTIVE
Build a fully autonomous LinkedIn content pipeline that requires human intervention ONLY at the approval step. Two stakeholders (owners) must be able to approve or reject a post via Telegram in a single tap. Everything else — art direction, publishing, engagement, and analytics — must be automated.

---

# REQUESTED PIPELINE (evaluate each layer)

## LAYER 1 — Art Director Agent
Input: raw Markdown from engine-content
Tasks:
  - Generate an image prompt optimized for LinkedIn (16:9 or 1:1)
  - Choose between: single image post, carousel (up to 5 slides), or text-only
  - Produce 3 hook variations for the opening line
  - Suggest 5-8 relevant hashtags based on topic and audience
Output: enriched content package (text + image prompt + format decision + hashtags)
Evaluate: feasibility with local LLM, image generation API options (Ideogram, DALL-E, Stable Diffusion local), carousel generation tools

## LAYER 2 — Approval Gate Agent
Input: enriched content package from Layer 1
Tasks:
  - Send formatted preview to both stakeholders via Telegram
  - Preview includes: hook, full post text, image prompt or image preview, hashtags, scheduled time
  - Stakeholder actions: Approve / Edit request / Reject
  - If both approve OR one approves and SLA of 24h passes: auto-advance to Layer 3
  - If rejected: send back to engine-content with rejection reason
Output: approved content package in Redis queue
Evaluate: Telegram bot architecture for 2-approver flow, Redis queue design for approval state

## LAYER 3 — Scheduler Agent
Input: approved content from Redis queue
Tasks:
  - Determine optimal publish time based on LinkedIn engagement patterns (Tue-Thu, 8-10am EST)
  - Manage posting frequency (max 1 post/day, min 3 posts/week)
  - Maintain a content calendar in PostgreSQL
  - Handle rescheduling on conflicts or rejected slots
Output: scheduled job with exact datetime, stored in PostgreSQL
Evaluate: cron architecture vs event-driven approach inside OpenClaw

## LAYER 4 — Publisher Agent
Input: scheduled job triggered from Layer 3
Tasks:
  - Authenticate with LinkedIn API (OAuth2, token refresh)
  - Publish text post, image post, or carousel (UGC Post API or Community Management API)
  - Attach image or carousel assets
  - Store post_id, publish timestamp, and content metadata in PostgreSQL
Output: published post record
Evaluate: LinkedIn API current limitations (2025/2026), carousel support status, rate limits

## LAYER 5 — Engagement Agent
Input: published post_id, runs on schedule every 30 minutes for first 4 hours post-publish
Tasks:
  - Fetch new comments via LinkedIn API
  - Classify comment intent: question / compliment / criticism / spam
  - Generate contextual reply using local LLM
  - Post reply (max 10-15 replies/day to respect rate limits)
  - Like relevant comments
  - Optionally: send automated welcome DM to new followers (evaluate feasibility)
Output: engagement log in PostgreSQL
Evaluate: LinkedIn API comment/reply endpoints, rate limit constraints, DM automation policy

## LAYER 6 — Analytics Agent
Input: post records from PostgreSQL, runs weekly (Sunday 8am)
Tasks:
  - Pull impressions, reactions, comments, shares, saves per post via LinkedIn API
  - Calculate engagement rate, reach, and performance score per post
  - Identify top-performing content patterns (topic, format, hook type)
  - Send weekly Telegram report to stakeholders (summary + top post + recommendations)
  - Feed performance score back to engine-content as context for next generation cycle
Output: analytics report in PostgreSQL + Telegram message
Evaluate: LinkedIn Analytics API availability for personal vs company pages, data freshness

---

# EVALUATION CRITERIA
For each layer, provide:
1. Technical feasibility score (1-5) with justification
2. Recommended implementation approach (tools, libraries, APIs)
3. Estimated build effort in days (solo developer)
4. Key risks and mitigations
5. Dependencies on other layers

---

# CONSTRAINTS
- All agents must run inside OpenClaw on the local Mac Mini M4 Pro
- LLM inference must use Ollama local models (qwen2.5-coder:14b preferred for code tasks)
- No cloud AI inference except for image generation (Ideogram API or equivalent)
- State persistence: Redis for short-lived queue state, PostgreSQL for permanent records
- Human interaction limited to: Telegram approval gate only
- Must comply with LinkedIn API Terms of Service (no fake engagement, no scraping)
- Budget: minimize paid APIs — prefer free tiers or open source alternatives

---

# DELIVERABLE FORMAT
Respond with:
1. Executive summary (3-5 sentences)
2. Layer-by-layer evaluation table (feasibility, effort, risk)
3. Recommended implementation sequence (which layers to build first)
4. Critical blockers or unknowns that need investigation before starting
5. Proposed PostgreSQL schema sketch (tables needed across all layers)
6. Proposed Redis key structure for approval queue and scheduling state
7. Suggested OpenClaw agent naming convention and inter-agent communication pattern