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
│   └── service/         # Domain services — pure business logic shared across aggregates
│                        # (e.g. a pricing engine used by both Order and Quote)
│                        # Must have zero dependencies on frameworks, infrastructure, or application code
├── application/
│   └── usecase/         # Use-case implementations: implement domain/port/in/ interfaces,
│                        # orchestrate domain objects, own transaction and validation boundaries
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

**Dependency rule**: `domain/` must never import from `adapter/` or `application/`. `application/` must never import from `adapter/`.
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
  if (!value) throw new InvalidUserIdError();
  return brand<string, 'UserId'>(value);
}
```

`InvalidUserIdError` extends `DomainError` (see [Domain model](#domain-model)). Value Object factories follow the same domain error rule as the rest of the domain layer.

- **Named exports only** — never `export default`.
- **Prefer immutability** — avoid using the `let` keyword unless strictly required.
- **Prefer pure functions** — domain functions take input and return output; they do not perform I/O, call infrastructure, read the clock, or log. State transitions are modelled as functions that return the new state rather than mutating in place. This is a strict requirement in the domain model.

---

## Domain model

- Aggregates expose state through meaningful domain methods, not raw setters.
- Aggregates reference each other by ID (branded type), never by object reference.
- Domain errors are distinct classes extending a base `DomainError`, never generic `Error`.
- Domain errors may carry structured data when the caller needs it to make a decision — prefer typed fields over embedding context in the message string.
- Domain errors are **not** exceptions in the exceptional sense; they are expected outcomes of business rules. Catch and handle them explicitly in the application layer.
- Never catch domain errors inside the domain itself — let them propagate to the boundary that knows how to react.

```typescript
// src/domain/model/errors.ts
export class DomainError extends Error {
  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
  }
}

// Carry structured data when the caller needs it
export class InsufficientStockError extends DomainError {
  constructor(
    readonly productId: ProductId,
    readonly requested: number,
    readonly available: number,
  ) {
    super(`Insufficient stock for product ${productId}: requested ${requested}, available ${available}`);
  }
}

export class OrderAlreadyConfirmedError extends DomainError {
  constructor(readonly orderId: OrderId) {
    super(`Order ${orderId} has already been confirmed`);
  }
}
```

Mapping to HTTP status codes happens in the adapter layer only, never in domain or application:

```typescript
// src/adapter/in/http/errorMapper.ts
export function toHttpStatus(error: DomainError): number {
  if (error instanceof InsufficientStockError) return 409;
  if (error instanceof OrderAlreadyConfirmedError) return 409;
  return 422;
}
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

### Transaction boundaries

Transaction management belongs to the application layer, never to the domain or the HTTP adapter.

**Preferred: Unit of Work port** — define a `UnitOfWorkPort` in `domain/port/out/`. It exposes the repositories involved in the transaction and a `commit()` method. The use case receives it via constructor injection; the adapter in `persistence/` provides the implementation. Use this approach whenever multiple repositories participate in the same transaction.

```typescript
// src/domain/port/out/UnitOfWorkPort.ts
export interface UnitOfWorkPort {
  readonly orderRepository: OrderRepository;
  readonly stockRepository: StockRepository;
  commit(): Promise<void>;
  rollback(): Promise<void>;
}

// src/application/usecase/PlaceOrderUseCase.ts
export class PlaceOrderUseCase implements PlaceOrderPort {
  constructor(private readonly uow: UnitOfWorkPort) {}

  async execute(command: PlaceOrderCommand): Promise<PlaceOrderResult> {
    try {
      const result: PlaceOrderResult =
        // ... domain logic using uow.orderRepository, uow.stockRepository ...
        {} as PlaceOrderResult;
      await this.uow.commit();
      return result;
    } catch (error) {
      await this.uow.rollback();
      throw error;
    }
  }
}
```

**Acceptable alternative: `TransactionPort`** — for use cases that touch a single repository and want less ceremony, define a `TransactionPort` with a `run<T>(work: () => Promise<T>): Promise<T>` method. The use case wraps its logic in the callback; the port implementation handles begin/commit/rollback.

```typescript
// src/domain/port/out/TransactionPort.ts
export interface TransactionPort {
  run<T>(work: () => Promise<T>): Promise<T>;
}
```

**Never use framework middleware** (e.g. a Fastify or Express hook) to open/close transactions. This hides a critical consistency boundary from the application layer and makes it impossible to test without the framework.

---

## Domain events

Aggregates may produce domain events as part of a state transition. Events are value objects defined in `domain/model/` — plain data, past tense, immutable.

```typescript
// src/domain/model/events.ts
export type OrderPlacedEvent = {
  type: 'OrderPlaced';
  orderId: OrderId;
  customerId: CustomerId;
  occurredAt: Date;
};
```

**Aggregates do not publish events.** They return events as data (in the use-case result, or collected on the aggregate). The application layer publishes them after a successful `commit()`, via an `EventPublisherPort` in `domain/port/out/`.

**Inbound message consumers** belong in `adapter/in/messaging/`. They contain no business logic. The translation they perform depends on the nature of the incoming message:

- **Instruction from another service** (e.g. `ShipOrder`) → translate to a **Command** and invoke the corresponding use case. Commands express intent and can be rejected.
- **Fact from another bounded context** (e.g. `PaymentConfirmed`) → translate to an **internal domain event** and route to the appropriate handler. Domain events represent something that already happened and carry no expectation of rejection.

Never pass raw external message payloads into the domain or application layer.

---

## HTTP adapter rules

- No business logic in route handlers.
- Validate input schema with **Zod** before calling the use case.
- Map domain errors to HTTP status codes in the adapter layer only.

---

## Dependency wiring

The `config/` directory is the composition root. It is the only place that instantiates concrete classes and wires them together. Nothing outside `config/` calls `new` on an adapter or use-case implementation.

A `PrismaClient` (or equivalent connection pool) is a singleton — one instance shared across the application. A `UnitOfWork`, however, holds per-transaction state and **must be created fresh for each request**. Inject a factory function rather than a shared instance:

```typescript
// src/config/container.ts
const db = new PrismaClient();

// Factory: called once per request, produces an isolated UoW
const uowFactory = (): UnitOfWorkPort => new PrismaUnitOfWork(db);
const placeOrder: PlaceOrderPort = new PlaceOrderUseCase(uowFactory);

export const container = { placeOrder };
```

```typescript
// src/application/usecase/PlaceOrderUseCase.ts
export class PlaceOrderUseCase implements PlaceOrderPort {
  constructor(private readonly uowFactory: () => UnitOfWorkPort) {}

  async execute(command: PlaceOrderCommand): Promise<PlaceOrderResult> {
    const uow = this.uowFactory(); // isolated transaction context per call
    // ...
    await uow.commit();
    return result;
  }
}
```

Route handlers and message consumers receive the `PlaceOrderPort` interface — they never import `PlaceOrderUseCase` directly. If the team adopts a DI container later, it lives in `config/` and the rest of the codebase doesn't change.

---

## Logging

Logging is infrastructure. **`domain/` must never log** — it has no knowledge of loggers, log levels, or structured fields.

- **Adapter layer** (`adapter/`) is the natural home for most logging: inbound request/response, outbound call durations, infrastructure errors.
- **Application layer** (`application/usecase/`) may log at key use-case boundaries (start, success, handled domain error) if needed, but only via an injected `LoggerPort` — never by importing Pino directly.
- **`main.ts` / `config/`** configures and instantiates the Pino logger and injects it where needed.

If the team only logs at the adapter boundary, no port is needed and Pino remains an adapter-layer detail. Introduce `LoggerPort` only when the application layer requires it:

```typescript
// src/domain/port/out/LoggerPort.ts
export interface LoggerPort {
  info(message: string, context?: Record<string, unknown>): void;
  warn(message: string, context?: Record<string, unknown>): void;
  error(message: string, context?: Record<string, unknown>): void;
}
```

Never log sensitive data (PII, payment details, credentials) at any layer.

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
