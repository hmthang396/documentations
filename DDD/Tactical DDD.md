# Documentation: Tactical Domain-Driven Design (DDD)

## Introduction

Domain-Driven Design (DDD) is a software design approach that focuses on accurately and effectively modeling the business domain. DDD is divided into two main parts:

- **Strategic DDD** — focuses on analyzing and dividing a large domain into bounded contexts, defining a ubiquitous language, and context mapping  
- **Tactical DDD** — provides concrete design patterns and building blocks for implementing the domain model in code

Tactical DDD offers specific tools (building blocks) to create a rich, expressive, and maintainable domain model. It helps developers avoid common pitfalls such as the **anemic domain model** (where domain objects are just data containers without behavior) by strongly emphasizing the combination of data and behavior within the same classes.

This document is written based on many years of practical experience applying DDD in real-world projects — from financial systems and insurance platforms to large-scale e-commerce and logistics systems.

## Core Building Blocks of Tactical DDD

### 1. Entities
- **Definition**: Objects that have a unique **identity** and can change state over time. Identity is independent of attribute values.
- **Characteristics**:
  - Mutable (state can change)
  - Compared by identity (usually an ID — UUID, auto-increment, etc.)
  - Usually represent core business concepts: Customer, Order, Invoice, Booking…
- **When to use**: When you need to track changes of an object throughout its lifecycle
- **Example** (TypeScript):
```typescript
export abstract class Entity {
  // Common entity logic (if any)
}

type OrderStatus = "Pending" | "Shipped";

export class Order extends Entity {
  private readonly id: string; // Guid → string (UUID)
  private status: OrderStatus;
  private readonly items: ReadonlyArray<OrderItem>;

  constructor(id: string, items: OrderItem[]) {
    super();
    this.id = id;
    this.items = Object.freeze([...items]); // enforce immutability
    this.status = "Pending";
  }

  get Id(): string {
    return this.id;
  }

  get Status(): OrderStatus {
    return this.status;
  }

  get Items(): ReadonlyArray<OrderItem> {
    return this.items;
  }

  ship(): void {
    if (this.status !== "Pending") {
      throw new Error("Only pending orders can be shipped.");
    }

    this.status = "Shipped";
  }
}
```

### 2. Value Objects
- **Definition**: Objects without conceptual identity — defined completely by their attributes. They are **immutable**.
- **Characteristics**:
  - Compared by value (structural equality)
  - No identity field
  - Usually small, descriptive concepts: Money, Address, DateRange, Email, Percentage…
- **When to use**: To eliminate primitive obsession and make domain concepts more expressive
- **Example** (TypeScript):
```typescript
export abstract class ValueObject {
  protected abstract getEqualityComponents(): unknown[];

  equals(other: ValueObject): boolean {
    if (!other || other.constructor !== this.constructor) {
      return false;
    }

    const thisComponents = this.getEqualityComponents();
    const otherComponents = other.getEqualityComponents();

    if (thisComponents.length !== otherComponents.length) {
      return false;
    }

    return thisComponents.every(
      (value, index) => value === otherComponents[index]
    );
  }
}

export class Money extends ValueObject {
  public readonly amount: number;   // decimal → number
  public readonly currency: string;

  constructor(amount: number, currency: string) {
    super();

    if (amount < 0) {
      throw new Error("Amount cannot be negative.");
    }

    if (!currency) {
      throw new Error("Currency is required.");
    }

    this.amount = amount;
    this.currency = currency;
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error("Currencies must match.");
    }

    return new Money(this.amount + other.amount, this.currency);
  }

  protected getEqualityComponents(): unknown[] {
    return [this.amount, this.currency];
  }
}
```

### 3. Aggregates
- **Definition**: A cluster of related Entities and Value Objects treated as a **single unit** for data consistency. Only one Entity serves as the **Aggregate Root**.
- **Key Rules**:
  - All external access must go through the Aggregate Root
  - Transaction boundary = Aggregate boundary
  - Maintain invariants within the Aggregate
  - Reference other Aggregates only by ID (never direct object reference)
- **Classic example**: Order (Aggregate Root) + OrderItems + ShippingAddress

### 4. Repositories
- **Purpose**: Provide collection-like interface for accessing and persisting **Aggregate Roots** (never individual Entities inside an Aggregate)
- **Important principles**:
  - Work only with Aggregate Roots
  - Hide all persistence details
  - Usually one Repository per Aggregate type
  - Do **not** expose IQueryable or complex query methods — keep domain logic in the domain

### 5. Domain Services
- **Definition**: Stateless operations that do not naturally belong to any single Entity or Value Object
- **When to use**:
  - Business logic spans multiple Aggregates
  - Complex calculations or transformations
  - Coordinating domain rules across entities
- **Examples**: PricingEngine, PaymentAuthorizationService, LoyaltyPointsCalculator

### 6. Domain Events
- **Definition**: Something important that happened in the domain that other parts of the system may care about
- **Common pattern**:
  - OrderPlaced
  - OrderShipped
  - PaymentReceived
  - CustomerReachedGoldTier
- **Purpose**: Enable loose coupling between different parts of the system (especially across bounded contexts)

### 7. Factories
- **Purpose**: Encapsulate complex object creation logic, especially for Aggregates
- **Benefits**:
  - Ensures invariants are satisfied from the moment of creation
  - Hides construction complexity
  - Improves readability

### 8. Modules / Packages
- Organize code by bounded context and by tactical pattern
- Common structure example:
```
src/
  Ordering/
    Domain/
      Aggregates/
      Entities/
      ValueObjects/
      DomainEvents/
      DomainServices/
      Repositories/
    Application/
    Infrastructure/
```

## Practical Application Summary

1. Start with **Ubiquitous Language** (from strategic DDD)
2. Identify **Aggregates** → define consistency boundaries
3. Model important behavior inside **Entities** and **Aggregates**
4. Use **Value Objects** aggressively to enrich the model
5. Create **Domain Services** only when logic doesn’t belong naturally to an Aggregate
6. Protect invariants through **Aggregate Roots**
7. Use **Repositories** to persist/load whole Aggregates
8. Publish **Domain Events** for important state changes
9. Keep infrastructure (database, external services) completely separated

## Quick Reference Table

| Concept            | Has Identity? | Mutable? | Lifecycle Tracking? | Typical Size     | Access Through     |
|--------------------|---------------|----------|----------------------|------------------|--------------------|
| Entity             | Yes           | Yes      | Yes                  | Medium–Large     | Directly / via AR  |
| Value Object       | No            | No       | No                   | Small            | By value           |
| Aggregate Root     | Yes           | Yes      | Yes                  | Large (cluster)  | Only entry point   |
| Domain Service     | No            | No       | No                   | Stateless        | Directly           |

If you would like a more detailed explanation of any particular pattern, real-world code examples in a specific language (C#, Java, TypeScript, Python, etc.), or how to combine Tactical DDD with modern architectures (Clean Architecture, Vertical Slice, Modular Monolith, Microservices…), feel free to ask!