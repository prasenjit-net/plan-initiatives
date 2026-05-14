# Plan Initiative — Project Guidelines

## Project Purpose

This is a **documentation-only** repository. It contains initiative plans, architecture designs, API specifications, and technical documentation for software projects. **There is no source code to implement, build, test, or run.**

## What Lives Here

```
initiatives/
  {initiative-name}/
    README.md               ← Initiative overview, goals, architecture summary
    IMPLEMENTATION_PLAN.md  ← Phases, components, file structure, config
    docs/
      ARCHITECTURE.md       ← Component design, data flow, diagrams
      API.md                ← HTTP API reference, endpoint specs, examples
      PROTOCOL.md           ← Protocol details, message formats, transport
      DEPLOYMENT.md         ← Deployment scenarios, Docker, Kubernetes
      DEVELOPMENT.md        ← Local setup, workflow, debugging
```

## Core Rules

1. **No code implementation.** Do not write Go, Python, or any other source code files. Do not create `go.mod`, `Dockerfile`, config files, or scripts unless they are documentation examples inside Markdown code blocks.

2. **No build or test commands.** There is nothing to compile, lint, or test. Do not suggest running `go build`, `npm install`, or similar commands.

3. **Markdown only.** All output is `.md` files. Maintain consistent heading hierarchy, ASCII diagrams, and table formatting matching the existing documents.

## Document Conventions

- **ASCII architecture diagrams** use `┌ ┐ └ ┘ ─ │ ├ └ ▼ ►` box-drawing characters — match the style in existing diagrams exactly.
- **Tables** use `|` pipe formatting with header separator rows.
- **Code blocks** inside Markdown use triple backticks with language hint (` ```go `, ` ```json `, ` ```bash `).
- **Tool names** in the MCP platform follow the `{service_id}::{tool_name}` namespace convention.
- **Version notes** for significant additions are marked with `> **vX.Y Additions:** ...` at the top of the relevant doc section.

## Initiative Structure

Each initiative has:
- A `README.md` as the single entry point (overview, goals, architecture, phases, tech stack, config, file structure, risk table)
- An `IMPLEMENTATION_PLAN.md` summarising phases, components, endpoints, config, and file layout
- A `docs/` folder with detailed specs per concern (architecture, API, protocol, deployment, development)

When adding or updating an initiative, keep **all files consistent** — a component added to one doc must be reflected in the others.

## Tone & Style

- Be precise and technical. These documents are read by engineers.
- Use present tense for current design decisions, future tense only for "Next Steps" sections.
- Rationale sections explain *why*, not just *what*.
- Avoid vague language ("some services", "various components") — name things explicitly.

## References

- MCP Platform initiative: `initiatives/mcp-platform/`
- Architecture: `initiatives/mcp-platform/docs/ARCHITECTURE.md`
- API spec: `initiatives/mcp-platform/docs/API.md`
