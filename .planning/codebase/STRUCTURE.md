# Codebase Structure

**Analysis Date:** 2026-06-09

## Directory Layout

```
code-reviewer-agent/
├── cinatra/
│   └── oas.json            # OAS Flow 26.1.0 agent definition (canonical runtime artifact)
├── skills/
│   └── code-review-methodology/
│       └── SKILL.md        # LLM skill: review checks, output contract, step-by-step instructions
├── .github/
│   └── workflows/
│       ├── ci.yml          # Standalone extracted-repo CI (build + kind-gates)
│       └── release.yml     # Release workflow
├── extension-kind-gate.mjs # Zero-dependency CI gate: OAS parse + retired-primitive scan
├── package.json            # Extension manifest (cinatra.kind: "agent", apiVersion, name, version)
├── tsconfig.json           # Standalone TypeScript config (targets src/ — no src/ exists yet)
├── .npmrc                  # npm/pnpm registry config (existence noted; contents not read)
├── LICENSE                 # Apache-2.0
└── README.md               # Project documentation
```

## Directory Purposes

**`cinatra/`:**
- Purpose: Cinatra platform sidecar directory — holds the agent's runtime artifact
- Contains: `oas.json` (the OAS Flow definition consumed by the WayFlow runtime)
- Key files: `cinatra/oas.json`

**`skills/`:**
- Purpose: Skill definitions that the Cinatra platform injects into LLM calls when `match_when.agent_id` matches
- Contains: One skill subdirectory per skill; each subdirectory has a `SKILL.md`
- Key files: `skills/code-review-methodology/SKILL.md`

**`skills/code-review-methodology/`:**
- Purpose: Encode the advisory code-review methodology for the agent's single LLM call
- Contains: `SKILL.md` with input/output spec, check categories, exclusion list, step-by-step instructions
- Key files: `skills/code-review-methodology/SKILL.md`

**`.github/workflows/`:**
- Purpose: GitHub Actions CI/CD pipeline for the extracted standalone repo
- Contains: `ci.yml` (build + kind-gates), `release.yml` (publish)
- Key files: `.github/workflows/ci.yml`, `.github/workflows/release.yml`

## Key File Locations

**Entry Points:**
- `cinatra/oas.json`: WayFlow runtime agent definition — `start_node` references component `start`
- `extension-kind-gate.mjs`: CI gate entry — `main()` invoked when run directly

**Configuration:**
- `package.json`: Extension manifest — declares `cinatra.kind`, `cinatra.apiVersion`, `name`, `version`, `description`
- `tsconfig.json`: TypeScript compiler config — targets `src/` with `outDir: "dist"` (no `src/` currently exists)
- `.npmrc`: Registry authentication config (existence only — contents not read)

**Core Logic:**
- `cinatra/oas.json`: Declares the three-node flow (StartNode → ApiNode → EndNode), all data/control flow edges, and `$referenced_components`
- `skills/code-review-methodology/SKILL.md`: All review check logic (naming, version hygiene, metadata, OAS shape, HITL hints)
- `extension-kind-gate.mjs`: `validateAgent()`, `runGate()`, `walkLlmStrings()`, `scanOasString()` — plus workflow gate helpers

**Testing:**
- No test files detected. The `tsconfig.json` is present for future TypeScript sources; `package.json` has no `test` script. CI runs `pnpm test --if-present` (no-op currently).

## Naming Conventions

**Files:**
- OAS artifact: `cinatra/oas.json` (fixed platform convention — always this path)
- Skill definitions: `skills/<skill-name>/SKILL.md` (kebab-case subdirectory, uppercase filename)
- Gate script: `extension-kind-gate.mjs` (kebab-case, `.mjs` ESM module)
- Workflow YAMLs: kebab-case (e.g., `ci.yml`, `release.yml`)

**Directories:**
- Skill subdirectories: kebab-case matching the skill's `name` frontmatter field (e.g., `code-review-methodology`)
- Platform sidecar: always `cinatra/` (fixed convention)

**OAS identifiers:**
- Flow `id`: kebab-case ending in `-flow` (e.g., `agent-code-reviewer-flow`)
- Component `id`s inside `$referenced_components`: stable kebab-case (e.g., `start`, `review`, `end`)
- DataFlowEdge `name`s: `<from>_to_<to>_<field>` pattern (e.g., `start_to_review_oasJson`)
- ControlFlowEdge `name`s: `<from>_to_<to>` pattern (e.g., `start_to_review`)

**Package names:**
- Agent packages: `@<scope>/<slug>-agent` pattern (e.g., `@cinatra-ai/code-reviewer-agent`)
- `metadata.cinatra.packageName` must match the npm package name implied by `packageSlug`

## Where to Add New Code

**New review check:**
- Add the check description and instructions to `skills/code-review-methodology/SKILL.md` under "What to check"
- No OAS changes required unless inputs change

**New input to the agent:**
- Add to `inputs` array at Flow level in `cinatra/oas.json`
- Add matching input to `$referenced_components.start.inputs`
- Add a DataFlowEdge in `data_flow_connections` from `start` to `review`
- Add matching input to `$referenced_components.review.inputs`

**New CI gate check:**
- Add to `extension-kind-gate.mjs` — extend `validateAgent()` for agent checks or add a new exported function
- Reference the new function from `runGate()`

**New skill:**
- Create `skills/<skill-name>/SKILL.md` with required frontmatter (`name`, `description`, `match_when`)
- No other files needed — the platform discovers skills by directory convention

**TypeScript source files (future):**
- Place under `src/` — `tsconfig.json` already points `rootDir` there
- Output will go to `dist/` (excluded from git via standard `.gitignore` conventions)

## Special Directories

**`cinatra/`:**
- Purpose: Platform-required sidecar holding the agent's OAS runtime artifact
- Generated: No — hand-authored OAS JSON
- Committed: Yes — this is the primary deployable artifact

**`skills/`:**
- Purpose: Skill definitions injected into LLM calls by the Cinatra platform
- Generated: No — hand-authored Markdown with YAML frontmatter
- Committed: Yes

**`.github/`:**
- Purpose: GitHub Actions workflows for CI/CD
- Generated: Partially — `ci.yml` was materialized by the extraction script; `release.yml` is maintained manually
- Committed: Yes

---

*Structure analysis: 2026-06-09*
