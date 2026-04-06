# Supervision Rules — Node.js / TypeScript

These rules apply to every AI-assisted session in a Node.js/TypeScript project.
They extend the organisation-wide rules in `CLAUDE.md`.

---

## Rules requiring explicit approval before proceeding

- Any modification to a public API (REST endpoints, event contracts, inter-service interfaces)
- Introduction of a new external dependency
- Changes to the package structure or module boundaries
- Any code touching authentication, authorisation, or sensitive data
- Changes to CI/CD configuration, Dockerfile, or infrastructure
- New architectural components (new adapters, new use cases, new aggregates, new domain services)

---

## Code quality — non-negotiable

- No `any`. If the type is unknown, use `unknown` with explicit narrowing.
- No `export default` — always named exports.
- No domain logic outside of `src/domain/` — this includes HTTP handlers, persistence adapters, and application use cases. Use cases orchestrate domain objects; they do not contain business rules.
- No `console.log` in production code — use Pino structured logging.
- No `catch` blocks that silently swallow errors.
- Functions and methods: maximum 30 lines. Classes: maximum 200 lines.

---

## Naming

- All source files: PascalCase for types, aggregates, ports, use cases, and adapters.
- Files in `src/domain/model/` and `src/domain/port/in/`: PascalCase (e.g. `OrderLine.ts`, `PlaceOrderPort.ts`).
- Files in `src/domain/port/out/`: PascalCase + `Port` suffix (e.g. `OrderRepositoryPort.ts`, `EventPublisherPort.ts`).
- Use-case implementations in `src/application/usecase/`: PascalCase + `UseCase` suffix (e.g. `PlaceOrderUseCase.ts`).
- Repository implementations in `src/adapter/out/persistence/`: PascalCase + `Repository` suffix (e.g. `OrderRepository.ts`).
- No generic names: `data`, `info`, `manager`, `helper`, `utils` are forbidden without explicit domain qualification.

---

## Testing

- Every new use case must have unit tests covering at least the happy path and the primary failure path.
- Domain errors must be tested explicitly — not just `rejects.toThrow()` but `rejects.toThrow(SpecificErrorClass)`. When the error carries structured data, assert the typed fields too, not only the error class.
- No test mocks internal collaborators — tests exercise behaviour through public interfaces only.

---

## Composition root

- Only `src/config/` may instantiate concrete classes and wire dependencies. Nothing outside `src/config/` calls `new` on an adapter or use-case implementation.
- Use-cases that own a `UnitOfWork` must receive a factory function (`() => UnitOfWorkPort`), not a shared instance, to guarantee an isolated transaction context per request.

---

## Security

- No secrets, API keys, or credentials in source code.
- User input validated at system boundaries (HTTP handler, message consumer) with Zod before entering the domain.
- Dependencies added only when strictly necessary; always declare version and licence.
