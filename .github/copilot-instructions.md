# Copilot Instructions (Fork-Specific)

## OpenClaw Questions — Official Docs First

When answering questions about OpenClaw (features, configuration, usage, APIs, architecture):

1. **Fetch official docs first** — check `https://docs.openclaw.ai/` (use `web_fetch`) for the relevant page before responding.
2. **Cross-check with source code** — verify the docs claim against the actual codebase (`src/`, `docs/`, config files).
3. **Official docs take precedence** — if docs and code disagree, present the docs answer as canonical and flag the discrepancy.
4. **Highlight discrepancies** — if the code behavior differs from official documentation, explicitly call it out (e.g., "⚠️ Discrepancy: docs say X, but the code does Y").
