# External Integrations

**Analysis Date:** 2026-06-09

## APIs & External Services

**Cinatra LLM Bridge:**
- Service: Cinatra internal `/api/llm-bridge` endpoint
  - Used by: `review` ApiNode in `cinatra/oas.json`
  - URL template: `{{CINATRA_BASE_URL}}/api/llm-bridge`
  - Auth: platform-injected (`CINATRA_BASE_URL` resolved by the Cinatra runtime at execution time)
  - Preferred provider: `openai`
  - Preferred model: `gpt-5.5`
  - Payload: POST with `system`, `user`, `agent_id`, `agent_run_id`, `cinatra_llm` fields

**Cinatra Marketplace / Registry:**
- Service: `registry.cinatra.ai` — publish target for extension releases
  - Triggered via: GitHub Release event → `.github/workflows/release.yml`
  - Reusable workflow: `cinatra-ai/.github/.github/workflows/reusable-extension-release.yml@main`
  - Auth: `CINATRA_MARKETPLACE_VENDOR_TOKEN` org secret (injected via `secrets: inherit`)

## Data Storage

**Databases:**
- Not applicable — this agent performs pure shape inspection of OAS JSON delivered inline. No database reads or writes.

**File Storage:**
- Local filesystem only — `extension-kind-gate.mjs` reads `package.json` and `cinatra/oas.json` from the package root at CI time. No remote file storage.

**Caching:**
- None

## Authentication & Identity

**Auth Provider:**
- Platform-managed — the Cinatra runtime resolves `{{CINATRA_BASE_URL}}` and authenticates requests to `/api/llm-bridge` transparently. No auth logic lives in this repo.

**Release Auth:**
- `CINATRA_MARKETPLACE_VENDOR_TOKEN` GitHub org secret, passed via `secrets: inherit` in `.github/workflows/release.yml`
- `id-token: write` + `attestations: write` permissions granted for build-provenance attestation (SLSA)

## Monitoring & Observability

**Error Tracking:**
- Not detected — no error-tracking SDK imported

**Logs:**
- Agent findings returned as structured JSON output: `{"findings": [...]}` with `ReviewFinding` objects (fields: `code`, `severity`, `warning|suggestion`, `message`, `location`, `source`)
- CI gate (`extension-kind-gate.mjs`) writes violations to stdout/stderr; exits 0 (clean) or 1 (violations)

## CI/CD & Deployment

**Hosting:**
- Cinatra Marketplace (registry.cinatra.ai) — publish on GitHub Release tag matching `v<package.json.version>`

**CI Pipeline:**
- GitHub Actions
  - `.github/workflows/ci.yml` — runs on push/PR to `main`; steps: checkout → Node 24 → corepack → first-party dep classification → conditional install → typecheck → test → pack dry-run → agent OAS validation gate (`extension-kind-gate.mjs`)
  - `.github/workflows/release.yml` — runs on GitHub Release published or manual `workflow_dispatch` against a tag; delegates entirely to reusable workflow in `cinatra-ai/.github`

## Environment Configuration

**Required env vars:**
- `CINATRA_BASE_URL` — resolved by the Cinatra runtime at agent execution; not a repo-level secret
- `CINATRA_MARKETPLACE_VENDOR_TOKEN` — GitHub org secret required for release workflow

**Secrets location:**
- GitHub org-level secrets (not stored in this repo)
- `.npmrc` present — sets `auto-install-peers=false` only; no auth tokens

## Webhooks & Callbacks

**Incoming:**
- GitHub Release event (`types: [published]`) triggers `.github/workflows/release.yml`

**Outgoing:**
- Agent POST to `{{CINATRA_BASE_URL}}/api/llm-bridge` at runtime (defined in `cinatra/oas.json` `review` ApiNode)
- Reusable release workflow submits to marketplace (extension-submit-for-review → approve → promotion saga) — implementation in `cinatra-ai/.github`, not this repo

## Skills / Prompt Injection

**Skill delivered at runtime:**
- `skills/code-review-methodology/SKILL.md` — matched on `agent_id: "@cinatra-ai/code-reviewer-agent"` and injected into LLM context by the Cinatra skill-routing layer. This is the sole source of the agent's review methodology; no external prompt registry or vector store is used.

---

*Integration audit: 2026-06-09*
