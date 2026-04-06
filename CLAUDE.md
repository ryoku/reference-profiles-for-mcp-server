# CLAUDE.md — Reference Profiles for MCP Server

## Project purpose

This repository is the **content backing** for an MCP server that coaches architecture and software design to coding assistants. The MCP server reads this repo at runtime and uses its content to guide, review, and generate code that conforms to established architecture standards for each supported stack.

## Repository structure

The repo is organised by stack. Each top-level folder represents one stack/architecture combination:

```
<stack-name>/          # e.g. node-typescript/
├── profile.json       # Machine-readable metadata for the MCP server
├── architecture.md    # Authoritative architecture guide for this stack
├── templates/         # Handlebars (.hbs) code generation templates
│   └── *.hbs
```

`node-typescript/` is the canonical reference for how a stack folder should be structured. Follow it when adding a new stack.

## Key files

| File | Purpose |
|---|---|
| `<stack>/profile.json` | IDs, expected directory layout, naming rules, and template paths consumed by the MCP server |
| `<stack>/architecture.md` | Human- and machine-readable architecture guide; the primary source of design coaching content |
| `<stack>/templates/*.hbs` | Handlebars templates used by the MCP server to scaffold conforming code |

## Conventions

- **`architecture.md` is the source of truth** for design decisions. Keep it internally consistent — examples must match the prose they illustrate.
- **`profile.json` must stay in sync** with `architecture.md`. If a directory or naming rule changes in the guide, update `profile.json` to match.
- **Templates live under `templates/`** and use Handlebars syntax. Variable names follow the conventions already established in existing templates (`pascalName`, `camelName`, …).
- Do not add root-level files that belong to a specific stack; keep all stack content inside its folder.

## Adding a new stack

1. Create a new top-level folder named `<stack>` (e.g. `python-fastapi`).
2. Add `profile.json`, `architecture.md`, and a `templates/` directory, following `node-typescript/` as the reference.
3. Ensure `profile.json` accurately reflects the expected directory layout and naming rules described in `architecture.md`.
