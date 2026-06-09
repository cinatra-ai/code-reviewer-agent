# Coding Conventions

**Analysis Date:** 2026-06-09

## Naming Patterns

**Files:**
- `kebab-case.mjs` for standalone gate scripts: `extension-kind-gate.mjs`
- `SKILL.md` in `skills/<kebab-case-name>/` for agent skill definitions
- `oas.json` placed under `cinatra/` directory for agent OAS contract

**Functions:**
- `camelCase` for all exported and internal functions: `parseArgs`, `validateAgent`, `validateWorkflow`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate`, `walkLlmStrings`, `scanOasString`, `wordBoundary`, `prefixOf`, `localOf`
- `main()` for the CLI entry point

**Variables:**
- `camelCase` for local variables and parameters: `packageRoot`, `oasPath`, `bpmnPath`, `allSidecars`
- `UPPER_SNAKE_CASE` for module-level constants: `LLM_VISIBLE_FIELDS`, `BANNED_PRIMITIVES`, `BANNED_TYPEHINTS`, `PRIMITIVE_PATTERNS`, `OBJECTS_LIST_CRM_RE`, `BPMN_MODEL_NS`, `WORKFLOW_PACKAGE_NAME_RE`

**Types:**
- No TypeScript types are declared in this repo's gate script (`extension-kind-gate.mjs` is plain `.mjs`). TypeScript is configured via `tsconfig.json` targeting a `src/` directory that does not yet exist — the repo is content-only (OAS + SKILL.md).

## Code Style

**Formatting:**
- No formatter config detected (no `.prettierrc`, `biome.json`, `.editorconfig`). Code style is consistent with 2-space indentation, double quotes for strings.

**Linting:**
- No ESLint or Biome config detected. The gate script has a header comment with a detailed scope explanation.

## Import Organization

**Order:**
1. Node built-ins using `node:` protocol prefix (e.g., `import { readFileSync, existsSync, readdirSync } from "node:fs"`)
2. No third-party imports (the gate is intentionally zero-dependency)

**Path Aliases:**
- None — the gate uses only Node built-in imports.

## Module Design

**Module type:** `"type": "module"` in `package.json` — all `.js`/`.mjs` files are ES modules.

**Exports:** Named exports for all testable units in `extension-kind-gate.mjs`:
- `parseArgs`, `validateAgent`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `validateWorkflow`, `runGate`

**CLI Guard Pattern:** The script guards `main()` with a direct-invocation check rather than a `if (require.main === module)` idiom (ESM equivalent):
```js
const invokedDirectly =
  process.argv[1] && resolve(process.argv[1]) === resolve(new URL(import.meta.url).pathname);
if (invokedDirectly) { main(); }
```

**Barrel Files:** Not applicable — single gate script.

## Function Design

**Size:** Functions are focused and single-purpose. `validateBpmnSanity` (~80 lines) is the largest — it handles XML tag-balance walking and namespace resolution in one pass. All others are ≤30 lines.

**Parameters:** Functions take primitive values (strings, objects) rather than class instances. No mutation of inputs.

**Return Values:**
- Validators return `string[]` (array of error messages) — empty array means pass.
- `runGate` returns `{ kind, errors }` object.
- `parseArgs` returns `{ packageRoot }` object.
- All validators are pure functions with no side effects.

## Error Handling

**Patterns:**
- File I/O wrapped in `try/catch`; errors are pushed into the `errors` array as string messages. Early `return errors` after I/O failures.
- `err instanceof Error ? err.message : String(err)` used consistently for safe error serialization.
- `process.exit(1)` for gate violations; `process.exit(0)` for pass — only in `main()`.
- The skill output contract specifies structured JSON error objects: `{ code, severity, message, location?, source }` with `severity` of `"warning"` or `"suggestion"` only (never `"blocker"`).

## Logging

**Framework:** `console.log` / `console.error` — no logging library.

**Patterns:**
- Success message on `stdout` via `console.log` with a `✓` prefix.
- Violation messages on `stderr` via `console.error` with a `✗` prefix and `•` bullet per violation.

## Comments

**When to Comment:**
- Extensive block comments at the top of every major section explaining the rationale and scope decisions (e.g., why the gate is zero-dependency, what the marketplace revalidates).
- Inline comments explaining non-obvious decisions (e.g., why `npx` is used instead of `pnpm dlx` for `tsc`).
- Contract-critical warnings noted prominently: e.g., the `{"findings":[...]}` envelope requirement is repeated multiple times in `SKILL.md` because violations cause a specific runtime failure.

**JSDoc/TSDoc:**
- No JSDoc comments. Public functions use leading `/** ... */` doc-style comments for description only (no `@param`/`@returns` tags): e.g., `/** Validate an agent extension at packageRoot. Pure: returns string[] errors. */`.

## Skill Output Contract

Per `skills/code-review-methodology/SKILL.md`, the agent must:
- Always return `{"findings": [...]}` — never a bare JSON array.
- Use only `"warning"` or `"suggestion"` severity levels.
- Include `"source": "agent-code-reviewer"` on every finding.
- Return `{"findings":[]}` when clean — not `{}` or `null`.

## CI / Package Conventions

**pnpm** is the package manager (`corepack pnpm`). Node ≥ 24.

**Package shape rules enforced by CI (`.github/workflows/ci.yml`):**
- Host-internal `@cinatra-ai/*` / `@cinatra/*` packages MUST be `peerDependencies` marked `peerDependenciesMeta.<pkg>.optional: true` — never `dependencies` or `devDependencies`.
- `cinatra.kind: "agent"` and `cinatra.apiVersion: "cinatra.ai/v1"` must be declared in `package.json`.
- Package name must follow `@<scope>/<slug>` pattern for agents.
- Workflow packages must match `@<scope>/<slug>-workflow`.

**Retired primitives** (banned from LLM-visible OAS strings): `lists_*`, `accounts_*`, `contacts_*` — use the `crm_*` facade instead.

---

*Convention analysis: 2026-06-09*
