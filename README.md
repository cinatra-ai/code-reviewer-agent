# Code Reviewer Agent

An advisory reviewer that checks new or updated Cinatra agent extensions for the kinds of quality issues a strict validator misses. It flags naming inconsistencies, missing version bumps, thin descriptions, and incomplete package metadata — surfacing warnings and suggestions you can act on before the extension ships, never blocking the build.

## Capabilities

- Catch naming inconsistencies between the package slug and its declared identity
- Flag missing version bumps before an extension is republished
- Surface incomplete package metadata that would hurt marketplace presentation
- Suggest description and contract improvements without blocking the build
- Return prioritized warnings and suggestions ready for the author to triage
