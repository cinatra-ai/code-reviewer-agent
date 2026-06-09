# Technology Stack

**Analysis Date:** 2026-06-09

## Languages

**Primary:**
- JavaScript (ESM) - `extension-kind-gate.mjs` (CI gate, zero-dependency, Node builtins only)
- TypeScript - `tsconfig.json` targets `src/**/*.ts` and `src/**/*.tsx` (no `src/` tracked yet; content-only extension)

**Secondary:**
- JSON - `cinatra/oas.json` (agent OAS Flow definition), `package.json`

## Runtime

**Environment:**
- Node.js 24 (pinned in `.github/workflows/ci.yml` via `actions/setup-node@v4`)

**Package Manager:**
- pnpm (via corepack — `corepack enable` in CI)
- Lockfile: not present (CI runs `pnpm install --no-frozen-lockfile`)

## Frameworks

**Core:**
- Cinatra OAS Flow 26.1.0 — agent runtime spec; the agent definition lives entirely in `cinatra/oas.json`

**Testing:**
- Not applicable — no test files tracked; CI runs `pnpm test --if-present` and skips gracefully

**Build/Dev:**
- TypeScript compiler (`tsc`) — configured via `tsconfig.json`; CI auto-resolves via `npx -y -p typescript tsc --noEmit` when no local `typescript` dependency is found
- `extension-kind-gate.mjs` — self-contained CI gate (no external deps; uses `node:fs`, `node:path` builtins only)

## Key Dependencies

**Critical:**
- None declared in `package.json` (`dependencies`, `devDependencies`, `peerDependencies` all absent). This is a content-only/source-mirror extension — all runtime dependencies are provided by the Cinatra monorepo.

**Infrastructure:**
- `cinatra.apiVersion: "cinatra.ai/v1"` — platform API contract declared in `package.json`
- `cinatra.kind: "agent"` — extension kind declared in `package.json`
- `cinatra.dependencies: []` — no inter-extension dependencies

## Configuration

**Environment:**
- `.npmrc` present — sets `auto-install-peers=false`. Never read contents beyond this note.
- No `.env` files detected.

**Build:**
- `tsconfig.json` — standalone strict TypeScript config; targets `ES2023`, `ESNext` modules, `bundler` module resolution, outputs to `dist/`, roots at `src/`; `jsx: "react-jsx"` included for future component support

## Platform Requirements

**Development:**
- Node.js 24+, corepack/pnpm
- No local install required for a content-only agent (source mirror pattern — `cinatra/oas.json` is the deployable artifact)

**Production:**
- Cinatra Marketplace / registry.cinatra.ai (publish via GitHub Release → reusable release workflow in `cinatra-ai/.github`)
- Requires `CINATRA_MARKETPLACE_VENDOR_TOKEN` org secret (wired via `secrets: inherit` in `.github/workflows/release.yml`)

---

*Stack analysis: 2026-06-09*
