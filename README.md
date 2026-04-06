# Reference Profiles for MCP Server

Content repository that backs an MCP server for architecture and software design coaching. The MCP server reads this repo at runtime to guide, review, and scaffold code that conforms to established architecture standards for each supported stack.

---

## How it works

The MCP server exposes the profile content as MCP tools that a coding assistant calls during a session:

| Tool | File it reads | Purpose |
|---|---|---|
| `get_architecture_guide` | `architecture.md` | Returns the full architecture guide for the active profile. |
| `get_supervision_rules` | `supervision-rules.md` | Returns the rules the agent must follow (what requires human approval, etc.). |
| `get_usage_guide` | `usage-guide.md` | Returns routing instructions — which tool to call for which task. |
| `get_layer_template` | `templates/*.hbs` | Renders a scaffolding template for a given layer and component name. |
| `validate_structure` | `profile.json` | Validates a project's directory layout and file naming against the profile rules. |
| `check_dependency_direction` | `profile.json` | Checks import-direction rules for a single file. |

The server selects a profile by matching the `MCP_ACTIVE_PROFILE` environment variable against the `id` field in `profile.json`, or automatically when only one profile is present.

### Stack sentinel

Consumer projects tell the agent which profile applies by adding a sentinel comment to their `CLAUDE.md`:

```markdown
<!-- mcp-stack: node-typescript -->
```

Replace `node-typescript` with the `id` of the profile defined in this repository.

---

## Repository structure

One top-level folder per supported stack:

```
<stack-name>/
├── profile.json           # Machine-readable metadata (directory layout, naming rules, template paths)
├── architecture.md        # Architecture guide — served by get_architecture_guide
├── supervision-rules.md   # Agent supervision rules — served by get_supervision_rules
├── usage-guide.md         # Tool routing instructions — served by get_usage_guide
└── templates/             # Handlebars (.hbs) scaffolding templates
    └── *.hbs
```

> All three Markdown files (`architecture.md`, `supervision-rules.md`, `usage-guide.md`) are **required**. The MCP server skips any profile directory that is missing one of them.

### Currently supported stacks

| Folder | Stack |
|---|---|
| `node-typescript/` | Node.js · TypeScript · Hexagonal Architecture |

---

## Adding a new stack

1. Create a top-level folder named after the stack (e.g. `python-fastapi/`).
2. Add all required files, using `node-typescript/` as the reference:

   | File | What to put in it |
   |---|---|
   | `profile.json` | Stack ID, expected source directories, file naming rules, and paths to all templates |
   | `architecture.md` | Full architecture guide — patterns, rules, and annotated code examples |
   | `supervision-rules.md` | Rules the agent must follow — what changes require human approval, what is forbidden, etc. |
   | `usage-guide.md` | Tool routing instructions — tells the agent which MCP tool to call for which task |
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
