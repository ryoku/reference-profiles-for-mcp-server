# Usage Guide — Node.js / TypeScript (Hexagonal)

This profile anchors coding agents to the hexagonal architecture conventions defined for Node.js / TypeScript services.

## When to activate this profile

Load this profile at the start of any session in a Node.js / TypeScript backend service that follows the hexagonal (ports and adapters) pattern.
The stack sentinel `<!-- mcp-stack: node-typescript -->` triggers automatic detection. This sentinel should be placed in the `CLAUDE.md` file of **the project that consumes this MCP server**.

## Routing instructions for agents

| Task | Tool to call |
|---|---|
| Add a new feature or use case | `get_layer_template` — retrieves the correct `.hbs` scaffold for the target layer |
| Validate directory structure or naming | `validate_structure` — reports violations with file, line, and suggestion |
| Check an import crosses layer boundaries | `check_dependency_direction` — enforces domain → application → adapter ordering |
| Read architecture reference | `get_architecture_guide` resource |
| Read supervision rules | `get_supervision_rules` tool or resource |

## Key conventions

- Business logic lives exclusively in `src/domain/`; no framework imports allowed there.
- Application services in `src/application/` orchestrate domain logic and call outbound ports.
- Adapters in `src/adapters/` implement inbound (HTTP, CLI) and outbound (DB, external APIs) ports.
- All cross-layer communication goes through explicitly typed port interfaces.
- New dependencies require explicit approval before introduction (see supervision rules).
