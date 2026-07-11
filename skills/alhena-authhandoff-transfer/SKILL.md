# Alhena: Prior-Authorization Handoff (Transfer)

**Date:** 2026-07-08
**Owner:** Samuel Quansah (Kojo Quansah), Principal AI Architect / Founding Engineer, 
**Next session focus:** Build the FastAPI routers (`src/api/routers/`)

## 1. What this project is

A production-grade, multi-agent Healthcare Prior Authorization platform. Architecture is deliberately domain-agnostic at the core so it generalizes beyond healthcare:

- `core/` — engine only: orchestration primitives, HITL gate mechanics, RAG interfaces, observability, security, config, audit. Contains **zero** references to prior auth / payer / patient.
- `domains/prior_auth/` — everything healthcare-specific: schemas, agent prompts/logic, the LangGraph graph wiring, EHR/FHIR + payer integrations.

Stack: **LangGraph** (orchestration) → **LangChain-style agent wrappers** → RAG/retrieval (payer policy store, clinical/EHR retriever) → integration layer (EHR/FHIR client, payer submission adapters) → FastAPI at the top. Cross-cutting: structlog, OpenTelemetry, PostgreSQL (pgvector), Pydantic v2, pydantic-settings.

## 2. Source of truth — do not duplicate, reference these

- **26-document architecture spec**, saved as `docs/00_...md` through `docs/25_...md` (approx.) during the design session. Key ones for the next task:
  - `docs/05_SYSTEM_ARCHITECTURE.md` — layered architecture diagram, core/domain split, case lifecycle data flow (section 5.3)
  - `docs/12_AGENT_CONTRACTS.md` — agent I/O contracts (e.g., `PriorAuthCase` schema, A1 Intake/Normalization contract)
  - `docs/13_PROMPT_ENGINEERING.md` — per-agent prompt design
  - `docs/19_ENGINEERING_DELIVERABLES.md` — the "reusable core" claim/spec
  - `docs/21_CLAUDE_SYSTEM_PROMPT.md` — supervised pairing-session system prompt
  - `docs/22_CODEX_SYSTEM_PROMPT.md` — stricter autonomous-agent system prompt with explicit **hard stops**: never touch `core/security/`, `core/audit/`, the `AUTONOMOUS_SUBMISSION_ENABLED` default, or any validator named `enforce`/`fail_closed`/`redact` without human review; tests must be written first and actually run before reporting success
- Original chat thread with the full build transcript: *"Multi-agent healthcare prior authorization platform design"* (search past chats for this title if you need to trace a specific decision).

## 3. Build status

Scaffolded and implemented so far (in a prior sandbox session — **see Known Gap below**):

```
src/core/{config, orchestration, agents, rag, hitl, observability, security, audit}
src/domains/prior_auth/{schemas, agents, graph, rag, integrations/submission_adapters, prompts}
src/api/{routers, middleware}   ← empty/scaffolded only, not yet implemented
tests/{unit/core, unit/domains, integration}
```

Confirmed implemented:
- `src/core/orchestration/checkpointer.py` — `get_checkpointer()`: `MemorySaver` for dev/test; single swap point for a durable Postgres-backed checkpointer in production (needed so a case paused at the HITL gate survives restarts / multi-day reviewer delays).
- `src/domains/prior_auth/schemas` — `PriorAuthCase` and related Pydantic models per `docs/12_AGENT_CONTRACTS.md` §12.2, using `SecretField` from `core/security/redaction`.
- Core/domain separation was verified as literal, not aspirational — a hypothetical future `domains/claims_review/` would reuse 100% of `core/` unchanged.

An earlier "ClaimSense" demo project was explicitly **not** reused — decision was to build the production platform fresh from the 26 docs.

## 4. Repositories (confirmed in git)

The implementation was pushed to git and is live in production, so the sandbox-reset risk noted in earlier drafts of this handoff no longer applies:

- **Frontend:** `github.com/samuelQUANSAH/careintel-brief-frontend`
- **Backend:** `github.com/samuelQUANSAH/careintel-brief-backend`
- **Production:** hosted on Vercel, custom domain `https://careintels.com/` (Cloudflare-proxied/masked in front of the Vercel deployment)

Note: these repos didn't resolve via public web search, which is expected and consistent with them being private — I haven't independently opened either repo to verify branch state or which parts of `src/api/routers/` (if any) already exist there. First step of the next session should be cloning both and diffing against the `src/api/` scaffold described in §3 before writing new router code, in case some of it was already started.

## 5. Next task — FastAPI routers

Build out `src/api/routers/`:
- Endpoints per `docs/05_SYSTEM_ARCHITECTURE.md` API layer: `/cases`, `/cases/{id}/approve|reject|edit`, `/webhooks`, `/health`
- Routers should be thin — call into the LangGraph orchestration layer (`PriorAuthGraph`), not embed business logic
- Follow `docs/22_CODEX_SYSTEM_PROMPT.md` hard stops if working with an autonomous coding agent: tests first, run the full suite before declaring done, run the PHI-in-logs scanner + prompt-injection red-team suite for anything touching agent/prompt/logging code

## 6. Suggested skills for the next session

From the `mattpocock/skills` package (`npx skills@latest add mattpocock/skills`):
- **`to-spec`** — turn the router requirements above into a formal spec before coding, since this is a fresh implementation task.
- **`tdd`** — the project's own system prompt (`docs/22_CODEX_SYSTEM_PROMPT.md`) mandates tests-first for any new endpoint.
- **`qa`** — run before declaring the routers done, given the compliance-sensitive domain.
- **`request-refactor-plan`** — only if routers end up needing to reshape any core/ interfaces (should be rare given the core/domain split).

## 7. Redactions

No API keys, credentials, or PHI/PII were present in the source material used to build this handoff. No redactions were necessary.
