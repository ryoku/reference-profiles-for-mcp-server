# Reference Profiles for MCP Server

Content repository that backs an MCP server for architecture and software design coaching. The MCP server reads this repo at runtime to guide, review, and scaffold code that conforms to established architecture standards for each supported stack.

---

## How it works

The MCP server surfaces three capabilities to a coding assistant:

1. **Coach** — answers architecture questions by referencing the relevant `architecture.md`.
2. **Review** — checks that proposed code or structure conforms to the rules in `architecture.md` and `profile.json`.
3. **Scaffold** — generates starter code by rendering the Handlebars templates in `templates/`.

---

## Repository structure

One top-level folder per supported stack:

```
<stack-name>/
├── profile.json       # Machine-readable metadata (directory layout, naming rules, template paths)
├── architecture.md    # Authoritative architecture guide for this stack
└── templates/         # Handlebars (.hbs) scaffolding templates
    └── *.hbs
```

### Currently supported stacks

| Folder | Stack |
|---|---|
| `node-typescript/` | Node.js · TypeScript · Hexagonal Architecture |

---

## Adding a new stack

1. Create a top-level folder named after the stack (e.g. `python-fastapi/`).
2. Add the three required artifacts, using `node-typescript/` as the reference:

   | File | What to put in it |
   |---|---|
   | `profile.json` | Stack ID, expected source directories, file naming rules, and paths to all templates |
   | `architecture.md` | Full architecture guide — patterns, rules, and annotated code examples |
   | `templates/*.hbs` | Handlebars scaffolding templates; use `pascalName` / `camelName` variables to match existing conventions |

3. Keep `profile.json` in sync with `architecture.md` at all times — if a directory or naming rule changes in the guide, update the JSON to match.

---

## Key conventions

- **`architecture.md` is the source of truth.** Prose and code examples must be internally consistent.
- **`profile.json` mirrors `architecture.md`.** Directory paths and naming patterns in the JSON must match the guide exactly.
- **Templates use Handlebars syntax.** Variable names (`pascalName`, `camelName`, …) follow the conventions in existing templates.
- **Stack content stays inside its folder.** Do not add stack-specific files at the repo root.

---

## Contributing

See [CLAUDE.md](CLAUDE.md) for guidance aimed at AI coding assistants working in this repo.
