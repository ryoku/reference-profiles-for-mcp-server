# Architecture Guide — Node.js / TypeScript (Hexagonal)

All Node.js backend services in this organisation follow **hexagonal architecture** (ports and adapters).
Business logic lives entirely in the `domain/` layer, which has zero dependencies on frameworks, databases, or HTTP.

---

## Directory structure

```
src/
├── domain/
│   ├── model/           # Types, aggregates, Value Objects — no external deps
│   ├── port/
│   │   ├── in/          # Use-case interfaces (driving ports)
│   │   └── out/         # Repository and external-service interfaces (driven ports)
│   └── service/         # Use-case implementations
├── application/
│   └── usecase/         # Orchestration, input validation, transaction boundaries
├── adapter/
│   ├── in/
│   │   ├── http/        # Fastify/Express route handlers
│   │   └── messaging/   # Event consumers (Kafka, SQS, …)
│   └── out/
│       ├── persistence/ # Repository implementations (Prisma, TypeORM)
│       └── external/    # HTTP clients toward external services
├── config/              # Dependency injection, configuration binding
└── main.ts              # Bootstrap only — no business logic
```

**Dependency rule**: `domain/` must never import from `adapter/` or `application/`.
Enforce this with TypeScript path aliases and the structure validator.

---

## TypeScript rules

- **No `any`** — use `unknown` with explicit narrowing.
- **No unjustified type assertions** (`as Foo`) — use type guards. The only exception is at validated boundaries such as branded-type factory/helpers, where the assertion is the mechanism that applies the brand after runtime checks.
- **`strict: true`** in `tsconfig.json`, no exceptions.
- **Branded types** for Value Objects to prevent primitive confusion:
  
```typescript
type Brand<T, TBrand extends string> = T & { readonly __brand: TBrand };
type UserId = Brand<string, 'UserId'>;
function brand<T, TBrand extends string>(value: T): Brand<T, TBrand> {
  return value as Brand<T, TBrand>;
}
function makeUserId(value: string): UserId {
  if (!value) throw new Error('UserId cannot be empty');
  return brand<string, 'UserId'>(value);
```

- **Named exports only** — never `export default`.

---

## Domain model

- Aggregates expose state through meaningful domain methods, not raw setters.
- Aggregates reference each other by ID (branded type), never by object reference.
- Domain errors are distinct classes extending a base `DomainError`, never generic `Error`.

```typescript
// src/domain/model/errors.ts
export class DomainError extends Error {
  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class InsufficientStockError extends DomainError {}
```

---

## Use cases

- One use case = one interface with a single `execute` method in `domain/port/in/`.
- Input and output are dedicated types (Command / Result).
- Return type is always `Promise<Result>` — never `void` on operations with output.

```typescript
// src/domain/port/in/PlaceOrderPort.ts
export interface PlaceOrderPort {
  execute(command: PlaceOrderCommand): Promise<PlaceOrderResult>;
}
```

---

## HTTP adapter rules

- No business logic in route handlers.
- Validate input schema with **Zod** before calling the use case.
- Map domain errors to HTTP status codes in the adapter layer only.

---

## Approved dependencies

| Purpose | Library |
|---|---|
| HTTP framework | Fastify |
| Schema validation | Zod |
| ORM | Prisma |
| Testing | Vitest |
| Logging | Pino |
| HTTP client | `undici` |

Any dependency outside this list requires explicit approval before addition.
