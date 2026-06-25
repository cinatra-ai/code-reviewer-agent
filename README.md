# Agent Code Reviewer

Advisory code-quality reviewer for Cinatra OAS Flow 26.1.0 agent definitions. Install from the Cinatra marketplace as `@cinatra-ai/code-reviewer-agent`, then invoke it through the Cinatra chat platform — no additional credentials are required beyond the host's `CINATRA_BASE_URL`. Pass the target agent's OAS body as `oasJson` (required), the package slug as `packageSlug` (optional), and any orchestrator hints as `reviewContext` (optional serialized JSON string, default `"{}"`). The flow returns a `findings` output string containing a JSON object `{"findings": [...]}` whose entries each carry a `severity` of `"warning"` or `"suggestion"` and an actionable `message`; it never emits blockers, which are owned by the deterministic lint gate. If `oasJson` cannot be parsed the response still arrives as a valid findings envelope so downstream data-flow edges extract correctly. For local development, run `node extension-kind-gate.mjs --package-root .` to verify the extension manifest before publishing.

## Works with

- Cinatra chat platform (OAS Flow 26.1.0 agent runner)
- Any Cinatra agent extension being authored or republished

## Capabilities

- Catch naming inconsistencies: Flow `id` kebab-case, `metadata.cinatra.packageName` vs `packageSlug`
- Flag missing version bumps before republishing to the marketplace registry
- Surface incomplete package metadata (`cinatra.apiVersion`, `cinatra.kind`, `description`)
- Check OAS contract conformance: `start_node` ordering, `agent_id` on `/api/llm-bridge` nodes, `cinatra_llm` per-node declarations
- Return prioritized `warning`/`suggestion` findings; never blocks the build
